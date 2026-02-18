# SQL Server + AX 2012 Print/Export Queue Latency Assessment (Updated with Current Index Inventory)

## Scope
This assessment focuses on whether increasing rowcount in `dbo.JAY_DOCEXPORTHISTQUEUE` can increase print-loop latency, and by which mechanism.

## What changed from prior assumptions
You provided the physical SQL index inventory. Two important updates:

1. `JAY_DOCEXPORTHISTQUEUE` **does already have** `I_109365INDEX_QUEUETIMEATTEMPTS` on `(PARTITION, DATAAREAID, ORIGQUEUETIME, ATTEMPTS)`.
2. `JAY_PRINTDEDUPETABLE` **does already have** `I_109372INDEX1` on `(PARTITION, DATAAREAID, SCANID, DOCUMENTRECID, DOCTYPE)`.

So the likely bottlenecks shift from "missing all hot-loop indexes" to "partially aligned indexes + lock/update patterns in loop".

---

## A) Hot-loop queries most likely to become O(N) / scan-heavy as rowcount grows

### 1) `TOP 1 ... ORDER BY OrigQueueTime` with `Status` filter (highest likelihood)
Hot loop methods (`hasPrintableWork`, `drainQueueChunk`) repeatedly query queue rows with predicates on status and attempts, ordered by queue time.

Observed code pattern:
- `where Status IN (Queued, FailedExport[, Processing])`
- `and Attempts < @maxAttempts`
- `order by OrigQueueTime`
- `firstOnly` (TOP 1)

Current index starts with `(OrigQueueTime, Attempts)`, **not `Status`**. Result:
- Optimizer may seek/sort by time but still evaluate status as residual predicate.
- As table grows and proportion of non-eligible statuses grows, engine reads more older rows to find first eligible row.
- That causes latency drift vs. rowcount even with a queue-time index present.

### 2) `recoverProcessingAsFailed()` inside each drain iteration (O(K) updates repeatedly)
`drainQueueChunk` invokes `recoverProcessingAsFailed(_queueType)` inside the while-loop. That method scans/updates all rows with `Status == Processing`.

Even if K (processing rows) is moderate, doing this repeatedly per loop pass amplifies total work and lock footprint. Under concurrency, this can dominate loop time and produce lock waits.

### 3) Reconcile probes by `ParentBatchRecId` + status/attempt logic
`reconcile()` iterates batches and probes queue table for any open rows by `ParentBatchRecId` with status/attempt predicates. You do have `ParentBatchRecId` index, but not with status/attempt in key; this can degrade with high batch fan-out and table growth.

### 4) Dedupe exists lookup (lower likelihood now)
`Jay_PrintDedupe::exists()` is called per document. Since an index exists, this is no longer top risk. However, key order is `(SCANID, DOCUMENTRECID, DOCTYPE)` while predicate logically filters `(SCANID, DOCTYPE, DOCUMENTRECID)`. It still works, but may not be optimal for all distributions.

---

## B) Specific index recommendations (key order + INCLUDE + filtered index guidance)

## 1) Primary queue-drain index (high value)
Create on `JAY_DOCEXPORTHISTQUEUE`:

- **Key**: `(PARTITION, DATAAREAID, STATUS, ORIGQUEUETIME, ATTEMPTS, RECID)`
- **INCLUDE**: `(DOCTYPE, DOCRECID, PARENTSCANID, PARENTBATCHRECID, SLICEID, QUEUETYPE, LASTATTEMPUTC)`

Why this order:
- `STATUS` first narrows to active states immediately.
- `ORIGQUEUETIME` supports FIFO retrieval.
- `ATTEMPTS` remains searchable for `< maxAttempts`.
- `RECID` stabilizes tie-breaking for TOP 1.

## 2) Batch reconcile index (targeted)
Create:
- **Key**: `(PARTITION, DATAAREAID, PARENTBATCHRECID, STATUS, ATTEMPTS)`
- **INCLUDE**: `(RECID)`

This optimizes repeated "does this batch still have open docs" probes.

## 3) Filtered index appropriateness
A filtered index is appropriate if active statuses are sparse and status domain is stable. Example filter:
- `WHERE STATUS IN (<Queued>, <Processing>, <FailedExport>)`

Caveat in AX environments:
- AOT schema sync does not manage SQL filtered indexes cleanly; maintain via post-sync DBA script and drift checks.

## 4) Dedupe index refinement (optional)
Existing index likely acceptable. If dedupe table grows materially, consider replacing/adding:
- `(PARTITION, DATAAREAID, SCANID, DOCTYPE, DOCUMENTRECID)`

so predicate order aligns exactly.

---

## C) Lock/concurrency risks and correctness-preserving mitigations

## 1) Non-atomic claim pattern (most important)
Current flow selects candidate row, then later acquires `forUpdate` to finalize. Multiple workers can race on same candidate.

Fix pattern:
- Atomically claim one row in single transaction:
  - select next eligible row with update lock semantics (`UPDLOCK`, `READPAST`, row-level preference)
  - immediately set `Status=Processing`, `SPID`, `LastAttemptUtc`
- Commit claim quickly.
- Print outside transaction.
- Finalize row status in short second transaction.

## 2) Excessive broad update in loop
Move `recoverProcessingAsFailed` out of per-row loop.
- Run once at cycle start, or every N seconds.
- Add staleness criterion (`LastAttemptUtc < now - threshold`) to avoid touching active rows.

## 3) Update frequency reduction
Avoid multiple queue-row updates for metadata-only changes unless needed for correctness (for example SPID/error writes can be coalesced).

## 4) Isolation strategy
If database allows `READ_COMMITTED_SNAPSHOT`, read-side blocking can drop significantly, while claim/update path stays lock-based for correctness.

---

## D) Minimal experiment to prove/disprove rowcount → latency

## Controlled variables
Hold constant:
- Worker count
- Batch composition (doc types, max batch size)
- Print pacing settings
- Hardware / DB options

## Independent variable
Increase only background rowcount in `JAY_DOCEXPORTHISTQUEUE` (e.g., 100k, 500k, 1M, 3M), while keeping active printable rows approximately constant.

## Metrics to capture per run
1. Throughput: docs/min and p50/p95 loop latency.
2. Query-level: elapsed ms + logical reads from `dm_exec_query_stats` for hot statements.
3. Waits: `LCK_*`, `PAGEIOLATCH_*`, `WRITELOG`, `CX*`, `SOS_SCHEDULER_YIELD` deltas.
4. Blocking/deadlocks: blocked process reports + deadlock graphs.
5. Actual plans for top hot statements.

## Interpretation
- If logical reads for `TOP 1` queries rise with total rowcount at constant active queue depth, mechanism is access path/selectivity drift.
- If waits shift toward `LCK_*` with concurrency, mechanism is claim/update contention.
- If plan shape flips between runs (seek→scan, spills/sorts), mechanism is plan instability/regression.

---

## E) Ranked likely causes + fastest/lowest-risk fixes first

1. **Status not leading in queue-drain access path** despite queue-time index.
   - Fast fix: add composite index with `STATUS` leading and covering includes.
2. **`recoverProcessingAsFailed` executed inside hot loop** causing repeated update scans.
   - Fast fix: move to periodic/start-of-cycle with stale-row predicate.
3. **Row claim race / lock contention from non-atomic selection-finalization.**
   - Medium fix: atomic claim pattern with lock hints semantics.
4. **Reconcile probes not fully covered by current key order.**
   - Fast-medium fix: `(ParentBatchRecId, Status, Attempts)` composite index.
5. **Dedupe key-order mismatch (secondary).**
   - Optional fix: reorder dedupe index to match predicate.
