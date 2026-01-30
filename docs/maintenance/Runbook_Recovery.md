# Incident Runbooks — Document Export / Print Orchestration

> **Scope:** These runbooks are based on the custom framework objects in this repo. Follow small, safe steps and prefer in‑AX recovery methods.

---

## 1) Stuck print cycle / export cycle

**Symptoms**
- Print cycle batch is not progressing, queues remain in `Processing` or `Queued`.
- `Jay_PrintGovernorState.IsPrinting = Yes` for an extended period.

**Likely causes**
- Print cycle batch died or is canceled, but governor flags were not cleared.
- Queue rows are stuck in `Processing` beyond the threshold.

**Diagnosis**
- Check `Jay_PrintGovernorState` (`IsPrinting`, `PendingPrintCycle`, `LastUpdateUtc`).
- Check queues for rows in `Processing` with old `LastAttemptUtc`.
- Check batch table for a PrintCycle job by caption `PrintCycle – Orchestrated Print Run`.

**Safe corrective actions (preferred in AX)**
1. Run **`Jay_ArbitrationService::DetectStuckPrinter()`** to cancel the active print batch (if any), clear governor flags, and reset queue rows to `FailedExport` so they can be retried safely.
2. Run **`Jay_PrintOrchestratorService::healPrintLocks()`** or **`ensureGovernorNotStuck()`** to clear `IsPrinting` if no print batch exists.
3. Use `notifyBatchReady` job (or `Jay_ArbitrationService::notifyBatchReady`) to schedule a print cycle if queue work exists.

**Verify**
- `Jay_PrintGovernorState.IsPrinting = No` and `PendingPrintCycle = No` after reset.
- A new PrintCycle batch is scheduled when work exists, and queue rows move to `Exported`/`FailedExport` with fresh `LastAttemptUtc`.

---

## 2) Batch job killed mid‑run

**Symptoms**
- Scan/Slice/Print batch shows `Canceled` or terminated.
- Scan/Slice/Batch control rows stuck in `Running` or `Pending` with no active batch.

**Likely causes**
- Batch server crash / manual cancel
- Governor flags left set (print) or slice/scan status not updated

**Diagnosis**
- Check for active batch jobs (by caption) matching Scan/Slice/Print jobs.
- Inspect `Jay_ScanCtrl.ScanStatus`, `Jay_SliceCtrl.SliceStatus`, `Jay_BatchCtrl.Status` for stuck states.

**Safe corrective actions (preferred in AX)**
1. Use **`Jay_ArbitrationService::DetectAndResolveState()`** to heal print locks, detect stuck slices/scans, and nudge restart where possible.
2. If print governor is stuck, use **`Jay_PrintOrchestratorService::healPrintLocks()`**.
3. If slices are stuck, use **`Jay_ArbitrationService::detectStuckSlices()`** (or `Jay_ArbitrationMaintenance::detectStuckSlices()` for warnings) to drive recovery.

**Verify**
- Stuck slices/scans transition to Running/Completed or are retried.
- No orphaned `IsPrinting` / `PendingPrintCycle` flags remain.

---

## 3) AOS restart during processing

**Symptoms**
- After AOS restart, queues stop progressing and print cycle does not start.
- Heartbeat job not running.

**Likely causes**
- Heartbeat job not bootstrapped or heartbeat lock left on.

**Diagnosis**
- Check `Jay_ExportRuntimeSettingsTable.HeartbeatLock` and `LastHeartbeatUTC`.
- Look for active batch jobs with caption `Jay Arbitration Heartbeat`.

**Safe corrective actions**
1. Run **`Jay_ArbitrationService::ensureHeartbeatAlive()`** to bootstrap heartbeat if missing.
2. If HeartbeatLock is stuck (Yes) and no heartbeat batch is running, clear the lock in AX via the runtime settings record.
3. Run **`Jay_ArbitrationService::DetectAndResolveState()`** to recover other states.

**Verify**
- Heartbeat job is scheduled and `LastHeartbeatUTC` updates on schedule.
- Print cycle can be scheduled when work exists.

---

## 4) Dedupe table blocking legitimate reprints

**Symptoms**
- Queue rows repeatedly marked `Skipped` without printing.
- A document will not reprint even after being re‑queued.

**Likely causes**
- `Jay_PrintDedupeTable` entry exists for the `ScanId` / `DocType` / `DocumentRecId`.

**Diagnosis**
- Check `Jay_PrintDedupeTable` for the relevant `ScanId` and document.

**Safe corrective actions (preferred in AX)**
1. Use **`Jay_PrintDedupe::deleteForScan(scanId)`** to remove dedupe entries for a scan.
2. Run **`Jay_ArbitrationService::cleanupDedupe()`** to remove dedupe entries for retired scans.

**Verify**
- Queue rows are processed normally and move to `Exported`.

---

## 5) Queue backlog / slow drain

**Symptoms**
- Queue rows build up faster than printing drains.
- Print cycle runs but throughput is low.

**Likely causes**
- Capacity settings too low (e.g., `PrintCapacityPerHour`, `PrintCadenceMs`).
- Print window constraints preventing printing.
- Excessive retries due to `MaxAllowedPrintAttempts` too low/high.

**Diagnosis**
- Check runtime settings in `Jay_ExportRuntimeSettingsTable` (`PrintCapacityPerHour`, `PrintCadenceMs`, `PrintWindowEnabled`, `PrintWindowStartTime/EndTime`).
- Confirm print cycle is running and governor flags are not stuck.

**Safe corrective actions**
1. Adjust print capacity settings (within approved limits) in `Jay_ExportRuntimeSettingsTable`.
2. Temporarily disable print window if it is blocking expected runs.
3. Use **`Jay_PrintOrchestratorService::reconcile()`** to close completed batches/slices/scans so new work can flow.

**Verify**
- Queue backlog decreases; queue row `LastAttemptUtc` updates.

---

## 6) Orphaned locks / governor thinks it’s already running

**Symptoms**
- `IsPrinting = Yes` but no print batch is running.
- New print cycles cannot schedule; `PendingPrintCycle` stays `Yes`.

**Likely causes**
- Print cycle crashed or AOS restarted while lock was held.

**Diagnosis**
- Check PrintCycle batch existence by caption `PrintCycle – Orchestrated Print Run`.
- Check `Jay_PrintGovernorState` flags.

**Safe corrective actions**
1. Run **`Jay_PrintOrchestratorService::ensureGovernorNotStuck()`** or **`healPrintLocks()`**.
2. If `PendingPrintCycle` is stuck, call **`clearPendingPrintCycle()`**.

**Verify**
- Governor flags are cleared and print cycle can schedule again.

---

## 7) “Nothing prints but jobs are running”

**Symptoms**
- PrintCycle batch is executing, but no rows progress.

**Likely causes**
- Dedupe skipping all rows.
- Queue rows are in `FailedExport` with attempts exhausted.
- Print window disabled or outside window.

**Diagnosis**
- Inspect queue row status/attempts (`Jay_DocExportProdQueue`, `Jay_DocExportHistQueue`).
- Check dedupe table (`Jay_PrintDedupeTable`).
- Check runtime settings (`PrintWindowEnabled`).

**Safe corrective actions**
1. If dedupe blocking is found, clear dedupe entries for the affected scan.
2. If attempts are maxed, adjust attempts (or re‑queue with reset) via AX processes.
3. Confirm print window settings allow printing now.

**Verify**
- Queue rows start moving to `Exported` or `FailedExport` with new `LastAttemptUtc`.

---

## 8) “Everything prints twice” (dedupe failure)

**Symptoms**
- Documents are printed multiple times even within a single scan.

**Likely causes**
- Dedupe entries not being written (e.g., `Jay_PrintDedupe::markPrinted()` not executing, exceptions swallowed).

**Diagnosis**
- Verify `Jay_PrintDedupeTable` rows for recent scans.
- Check for failed inserts or suppressed exceptions in print logs.

**Safe corrective actions**
1. Validate that `Jay_PrintOrchestratorService::drainQueueChunk()` is reaching `Jay_PrintDedupe::markPrinted()` after success.
2. Re‑run with logging enabled to confirm dedupe writes are successful.

**Verify**
- Dedupe rows are created and duplicates are skipped in subsequent passes.

---

# SQL‑level interventions (last resort)

> **Only after AX‑level recovery fails.** Do not run blanket delete/update scripts.

- **Governor flags**: Only touch the single row in `Jay_PrintGovernorState`. Clear `IsPrinting` and/or `PendingPrintCycle` if **no PrintCycle batch exists**.
- **Heartbeat lock**: Clear `Jay_ExportRuntimeSettingsTable.HeartbeatLock` if **no heartbeat batch exists** and `LastHeartbeatUTC` is stale.
- **Queue rows stuck in `Processing`**: Identify rows with old `LastAttemptUtc` and update **only those rows** to `FailedExport` to allow retry.

Always verify that parent control rows (`Jay_BatchCtrl`, `Jay_SliceCtrl`, `Jay_ScanCtrl`) still exist and are consistent before manipulating queue data.
