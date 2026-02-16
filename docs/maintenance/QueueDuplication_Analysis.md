# Queue Duplication Analysis (JAY_DOCEXPORTHISTQUEUE)

## Executive finding

The strongest code-level multiplier is in the fast enqueue path (`Jay_EnqueueService.bulkStageFromSlice`).
Per batch, it previously rebuilt `batchRecs` from the **entire** `_recIds` container (minus only the seed), then staged all of those rows again. This repeats across each batch and can inflate rows roughly by the number of batches in the slice.

For a slice with ~18.5k docs and a cap of 5k, batch count is ~4, which matches the observed ~4x pattern.

## Where queue rows are inserted

- `Jay_EnqueueService.safeEnqueueFromSlice` inserts one queue row with a guard (`select firstOnly`) for legacy/per-record enqueue.
- `Jay_EnqueueService.promoteStageToQueueByBatch` uses `insert_recordset` from temp staging tables into `Jay_DocExportHistQueue`/`Jay_DocExportProdQueue` and had no dedupe join in that bulk insert.
- `enqueueSlice` chooses fast vs legacy via runtime setting `EnqueueType`.

## Why STATUS=0 duplicates accumulate

Rows are inserted with `Status = Jay_ExportQueueStatus::Queued`. In your dataset, STATUS=0 aligns with unprinted queued work, consistent with enqueue-side multiplication rather than print-side retries.

## Slice execution/retry model

- `Jay_SliceProbeManager.runProbes` creates one `Jay_SliceCtrl` row per `(DocType, DocSubType, window)` for included scope entries.
- `Jay_SliceCtrlService.StartScanSliceProcessing` schedules pending/deferred/failed/partial slices (subject to retry limit), so slices can be attempted again.
- `Jay_EnqueueService.enqueueSlice` marks slices completed or failed and increments retry count on failure.

## DocType/SubType specificity hypothesis

Your hot combination `(DocType=1, DocSubType=1)` is Invoice/Part by enum values. If that population is the one that frequently exceeds the batch cap, it will be disproportionately affected by the fast-path batch multiplication bug.

## Change made in this branch

## Are primary queries the root cause?

Likely **not** for the 4x spike:

- Invoice/Part enumeration reads `CustInvoiceJour` directly (no child joins), so it does not naturally create fixed fan-out multiplicity by join explosion.
- The dominant deterministic multiplier was the fast batching bug (restaging nearly full container per batch).

That said, query overlap can still create occasional duplicates over time (for example, replay/retry windows). To harden this boundary, enqueue now de-duplicates enumerated RecIds before staging/enqueue (`uniqueRecIds(...)`) so query-side duplicates cannot inflate queue rows.

- Fixed fast-path batching so each batch stages only its own `[startIndex..endIndex]` chunk (minus the seed row already inserted safely), instead of re-staging nearly the whole slice each batch.
- Added `getBatchRecIds(...)` helper and switched `bulkStageFromSlice` to use it.

## Recommended next hardening (not yet implemented)

1. Add enqueue-time idempotency in `promoteStageToQueueByBatch` (`notexists join` on logical identity + queued status).
2. Add/verify a supporting index on hist queue for logical identity + status for performance.
3. Add a data-fix script to collapse existing duplicate STATUS=0 rows, keeping earliest row per identity.
4. Instrument per-slice metrics: distinct docs vs inserted rows, and multiplier alerting when >1.05x.
