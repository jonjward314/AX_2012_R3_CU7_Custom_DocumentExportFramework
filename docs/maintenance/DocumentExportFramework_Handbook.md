# Document Export / Print Orchestration Framework — User + Maintenance Handbook

## 1) System boundaries & roles

### What the framework **is responsible for**
- **Scanning / slicing / enqueueing work** into production and historical export queues (ScanCtrl → SliceCtrl → BatchCtrl → Queue rows). This is orchestrated by the Scan/Slice services and batch tasks (e.g., `Jay_ScanCtrlService`, `Jay_SliceCtrlService`, `Jay_EnqueueService`, `Jay_ScanCtrlBatch`, `Jay_SliceCtrlBatch`).
- **Print orchestration / pacing** of queued work, including pacing, single‑run governance, and reconciliation of Scan/Slice/Batch states via `Jay_PrintOrchestratorService`. It enforces a single print cycle at a time using the `Jay_PrintGovernorState` table (`IsPrinting`, `PendingPrintCycle`, `LastPrintScheduleUtc`).
- **Arbitration and heartbeat supervision**, via `Jay_ArbitrationService` / `Jay_ArbitrationBatch` heartbeat tasks, which call state recovery, cleanup, and re‑scheduling logic. The heartbeat lock is stored in `Jay_ExportRuntimeSettingsTable.HeartbeatLock`.
- **Dedupe protection** on prints using `Jay_PrintDedupeTable` (`Jay_PrintDedupe` handles insert/exists/delete).
- **Operational cleanup** (retention purge) using `Jay_ArbitrationService::CleanOldRecords()`.

### Who uses it, and what they do
- **Operations / IT (batch admins)**
  - Monitor batch jobs (Heartbeat, Scan, Slice, Print cycle)
  - Pause / resume enqueueing via runtime settings (Prod/Historical pause flags)
  - Run recovery tools (e.g., detect stuck, heal locks) and safe resets
- **Power users / support**
  - Trigger manual recovery jobs where approved (e.g., `notifyBatchReady` job)
  - Review queue/batch/slice/scan states for status and errors

## 2) Inventory of operational objects

### Batch jobs (RunBaseBatch)
- **`Jay_ArbitrationBatch`** — Orchestrates heartbeat, slice generation, and print cycle request processing. Uses `Jay_ArbitrationBatchFunctions` to branch into work (Heartbeat, GenerateScanSlices, RequestProcessPrintQueue, etc.).
- **`Jay_ScanCtrlBatch`** — Batch wrapper for scan operations (StartScan/Cleanup).
- **`Jay_SliceCtrlBatch`** — Batch wrapper for slice processing (e.g., `RequestProcessScanSlices`).

### Governor / orchestrator classes
- **`Jay_PrintOrchestratorService`** — print cycle governor (single‑run lock via `Jay_PrintGovernorState`, queue drain with pacing, reconciliation, and lock healing).
- **`Jay_ArbitrationService`** — arbitration engine that schedules work, maintains heartbeat, runs cleanup, and detects stuck work.
- **`Jay_ArbitrationMaintenance`** — diagnostic utilities for detecting stuck scans/slices/print cycles.

### Queue / control tables
- **Queues**
  - `Jay_DocExportProdQueue` (Production work queue)
  - `Jay_DocExportHistQueue` (Historical work queue)
- **Control / State**
  - `Jay_ScanCtrl` (scan lifecycle state)
  - `Jay_SliceCtrl` (slice lifecycle state)
  - `Jay_BatchCtrl` (batch lifecycle state)
  - `Jay_ScanScope` (scan scope definition)
  - `Jay_PrintGovernorState` (print governor lock)
  - `Jay_ExportRuntimeSettingsTable` (runtime governors, pause flags, heartbeat lock, cadence)
- **Dedupe & logging**
  - `Jay_PrintDedupeTable` (printed document dedupe)
  - `Jay_ArbitrationHistoryLog` (arbiter logging)

### “Kill / reset” utilities present in repo
- `Job_Jay_ClearAllExportTables` — **destructive** clear of all export/print tables (governor, queues, slices/scans/batches, logs). Use only in non‑prod or with explicit approval.
- `Jay_ArbitrationService::DetectStuckPrinter()` — resets stuck print queue rows, cancels active print batch, clears governor flags.
- `Jay_PrintOrchestratorService::healPrintLocks()` / `ensureGovernorNotStuck()` — releases print lock when no active batch exists.
- `Jay_ArbitrationMaintenance::runAll()` — detection alerts for stuck scans/slices/print.

## 3) State model (flags, locks, markers)

### Print governor (`Jay_PrintGovernorState`)
- **`IsPrinting`** — set in `Jay_PrintOrchestratorService::tryStartPrint()`, cleared in `endPrint()`, and can be cleared by recovery logic (e.g., `ensureGovernorNotStuck()`, `healPrintLocks()`, `DetectStuckPrinter()`).
- **`PendingPrintCycle`** — set in `Jay_PrintOrchestratorService::setPendingPrintCycle()` when scheduling a print cycle, cleared in `clearPendingPrintCycle()` and also cleared in recovery (`DetectStuckPrinter()`).
- **`LastPrintScheduleUtc`** — set when `PendingPrintCycle` is set.

### Heartbeat governor (`Jay_ExportRuntimeSettingsTable`)
- **`HeartbeatLock`** — set at start of `Jay_ArbitrationService::heartbeat()`, cleared in normal completion or exception handling.
- **`LastHeartbeatUTC`** — updated during heartbeat and on heartbeat bootstrap.

### Queue row markers (`Jay_DocExportProdQueue` / `Jay_DocExportHistQueue`)
- **`Status`** — set to `Queued` during enqueue staging; transitions to `Exported` or `FailedExport` in `Jay_PrintOrchestratorService::drainQueueChunk()`; can be reset from `Processing` to `FailedExport` by stuck‑print recovery (`DetectStuckPrinter()`).
- **`Attempts`** and **`LastAttemptUtc`** — updated each print attempt; used to cap retries (`MaxAllowedPrintAttempts`).
- **`SPID`** — set by `Jay_PrintOrchestratorService::setQueueSpid()` for tracing the SQL session.
- **`ErrorMessage`** — updated via `logQueueError()` / `updateErrorMessage()`.

### Scan / Slice / Batch lifecycle
- **`Jay_ScanCtrl.ScanStatus`** — scanned to decide if work is still open; set to `Completed` by reconciliation (`Jay_PrintOrchestratorService::reconcile()`).
- **`Jay_SliceCtrl.SliceStatus`** — used to arbitrate slice start, detect stuck slices, and mark completion during reconcile.
- **`Jay_BatchCtrl.Status`** — created as `Pending` during enqueue; set `Completed` by `Jay_PrintOrchestratorService::reconcile()` once all child queue rows are complete.

### Dedupe
- **`Jay_PrintDedupeTable` rows** — created by `Jay_PrintDedupe::markPrinted()`; consulted by `Jay_PrintOrchestratorService::drainQueueChunk()` to skip duplicate prints; removed by `Jay_PrintDedupe::deleteForScan()` or cleanup.

## 4) Safe operation quickstart (operators)

### Normal operation
1. Ensure runtime settings exist (`Jay_ExportRuntimeSettingsTable::findOrCreate()` is called by services).
2. Ensure heartbeat is running (`Jay_ArbitrationService::ensureHeartbeatAlive()` can bootstrap it).
3. Scan → slice → enqueue should flow automatically; print cycle is scheduled by `notifyBatchReady` / `notifyBatchReady` class logic when queue items exist.
4. Monitor queues for growth and slice/scan statuses for progress.

### Pause / resume
- **Pause enqueueing**: set `ProdEnqueuePause` and/or `HistEnqueuePause` in `Jay_ExportRuntimeSettingsTable`. `Jay_ArbitrationService::allowSliceStart()` uses these flags to defer slices.
- **Resume**: clear those pause flags; slices will be allowed to start again.

### Stop / restart
- **Stop**: cancel active batch jobs for print cycle or scan/slice batches. If a print cycle is stopped, run the recovery steps to clear the governor flags and reset stuck queue rows.
- **Restart**: allow heartbeat and arbitration to resume; optionally use `notifyBatchReady` (job) to schedule a print cycle if queues exist but no cycle is scheduled.

### What not to do (anti‑patterns)
- **Do not delete queue rows manually** without understanding parent `BatchCtrl` / `SliceCtrl` integrity; reconciliation expects those links.
- **Do not run `Jay_ClearAllExportTables` in production** except for complete system resets, as it deletes all state and queues.
- **Do not force‑set `IsPrinting` / `PendingPrintCycle` to `Yes`**; use the provided scheduling path to avoid duplicated print cycles.

## 5) Recovery & reset guardrails (overview)
- **Prefer in‑AX logic**: `Jay_ArbitrationService::DetectAndResolveState()`, `DetectStuckPrinter()`, `Jay_PrintOrchestratorService::healPrintLocks()`, and `Jay_ArbitrationMaintenance::runAll()`.
- **SQL changes only as last resort**: limit scope to stale records and only after validating they are truly orphaned (see runbooks).
- **Always verify**: after reset, confirm `IsPrinting = No`, `PendingPrintCycle = No`, and that queues are in a consistent status progression.

---

## 6) Appendix — Primary objects to know

| Object | Type | Purpose |
| --- | --- | --- |
| `Jay_PrintOrchestratorService` | Service | print governor, queue drain, reconcile, lock healing |
| `Jay_ArbitrationService` | Service | heartbeat, arbitration, cleanup, stuck detection |
| `Jay_ScanCtrlService` | Service | scan scheduling/execution |
| `Jay_SliceCtrlService` | Service | slice creation/processing |
| `Jay_EnqueueService` | Service | staging/queue creation |
| `Jay_PrintGovernorState` | Table | print governor flags (`IsPrinting`, `PendingPrintCycle`) |
| `Jay_ExportRuntimeSettingsTable` | Table | runtime settings, heartbeat lock, pause flags |
| `Jay_DocExportProdQueue` / `Jay_DocExportHistQueue` | Table | queued work with status/attempts/error |
| `Jay_ScanCtrl` / `Jay_SliceCtrl` / `Jay_BatchCtrl` | Table | scan/slice/batch lifecycle control |
| `Jay_PrintDedupeTable` | Table | printed dedupe markers |
