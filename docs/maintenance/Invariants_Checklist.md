# Maintenance Invariants Checklist — Document Export Framework

> Use this checklist during routine maintenance and incident response. Each invariant references concrete objects/fields in this repo.

## Print governor & heartbeat
- ✅ **At most one canonical print governor row** exists; use `Jay_PrintGovernorState::findOrCreate()` to enforce singleton.
- ✅ **`Jay_PrintGovernorState.IsPrinting` is `No` when no PrintCycle batch exists** (caption: `PrintCycle – Orchestrated Print Run`).
- ✅ **`Jay_PrintGovernorState.PendingPrintCycle` is `No` unless a print cycle is scheduled.**
- ✅ **`Jay_ExportRuntimeSettingsTable.HeartbeatLock` is `No` unless heartbeat is actively executing.**
- ✅ **`Jay_ExportRuntimeSettingsTable.LastHeartbeatUTC` advances on the expected interval.**

## Queue health
- ✅ **Queue rows in `Processing` have recent `LastAttemptUtc`.** Old `Processing` rows should be reset via `DetectStuckPrinter()`.
- ✅ **Attempts for queue rows (`Attempts`) do not exceed `MaxAllowedPrintAttempts` without being terminal (`FailedExport`).**
- ✅ **Queue rows reference valid `ParentBatchRecId`, `SliceId`, and `ParentScanId`.**

## Scan / Slice / Batch lifecycle
- ✅ **Each `Jay_BatchCtrl` row references a valid `ParentSliceRecId`.**
- ✅ **Batches with no open queue rows transition to `Jay_BatchStatus::Completed`.**
- ✅ **Slices with no open batches transition to `Jay_SliceStatus::Completed`.**
- ✅ **Scans with all slices completed transition to `Jay_ScanStatus::Completed`.**

## Dedupe correctness
- ✅ **`Jay_PrintDedupeTable` should contain one entry per successfully printed document per scan.**
- ✅ **Dedupe entries for retired scans are removed (via `cleanupDedupe()` or retention cleanup).**

## Pause / resume flags
- ✅ **`ProdEnqueuePause` / `HistEnqueuePause` should be `No` in normal operation.**
- ✅ **When pause flags are `Yes`, slices are expected to be deferred, not running.**

## “If a job is killed, clear” checklist
- ✅ If a PrintCycle batch is killed, clear:
  - `Jay_PrintGovernorState.IsPrinting`
  - `Jay_PrintGovernorState.PendingPrintCycle`
  - Any `Jay_DocExport*Queue` rows stuck in `Processing` beyond threshold
- ✅ If a heartbeat batch is killed, clear:
  - `Jay_ExportRuntimeSettingsTable.HeartbeatLock`
- ✅ If a slice batch is killed, ensure:
  - `Jay_SliceCtrl.SliceStatus` is corrected (Running → Pending/Deferred/Running) based on child batch presence
- ✅ If a scan batch is killed, ensure:
  - `Jay_ScanCtrl.ScanStatus` is consistent with child slice states

