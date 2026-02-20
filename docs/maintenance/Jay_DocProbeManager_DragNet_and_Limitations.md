# Jay_DocProbeManager: Query Drag-Net Behavior, Blind Spots, and Assumptions

## Purpose
This document explains how `Jay_DocProbeManager` identifies candidate records (the "drag net") and where each net can miss records or include unexpected ones.

The class exposes three probe styles:
- **quickProbe***: boolean existence check (`firstOnly`)
- **fullProbe***: count check (`count(RecId)`)
- **enumerate***: returns `RecId` container for downstream processing

All probes are dispatched by doc type (and sub type where relevant).

---

## Global drag-net semantics

### 1) Time window shape is half-open
For nearly all probes, date matching is:
- `>= _fromUtc`
- `< _toUtc`

So records exactly equal to `_toUtc` are excluded.

### 2) Invalid windows short-circuit
If `_fromUtc >= _toUtc`, probes return false/0/empty container.

### 3) Company filter
All queries enforce `DataAreaId == curext()`.

### 4) Stateful-table scan assumption
Most nets are based on **CreatedDateTime/ModifiedDateTime of current header tables**, not event history tables. This means "business event happened" and "record touched" are treated as equivalent in most nets.

---

## Per-document drag-net definitions

## Purchase Order (`PurchTable`)
Record is captured when all are true:
- Current company
- `PurchStatus` in `Backorder | Received | Invoiced`
- `CreatedDateTime` OR `ModifiedDateTime` inside window

### Blind spots / assumptions
- Canceled/other statuses are intentionally invisible.
- Any modification in-window can re-surface an old PO.

---

## Confirmation (`CustConfirmJour`) by subtype
Record is captured when all are true:
- Current company
- `CreatedDateTime` OR `ModifiedDateTime` inside window
- Subtype-specific `CustGroup` rules:
  - **Part**: `Internal | Part | Part_DPU`, except `Internal + MCASHIPPINGINSTRUCTIONS == 'Unit'`
  - **Unit**: `Unit | Unit_PO | (Internal + MCASHIPPINGINSTRUCTIONS == 'Unit')`
  - **Vendor**: `Vendor`

### Blind spots / assumptions
- Classification depends on current `CustGroup` + internal flags, not a historical subtype snapshot.
- Internal orders can shift net membership when shipping instructions change.

---

## Invoice (`CustInvoiceJour`) by subtype
Record is captured when all are true:
- Current company
- `CreatedDateTime` OR `ModifiedDateTime` inside window
- Subtype-specific rules:
  - **Part**: `Part | Part_DPU | (Internal + MCAAMDEVICEBRANDID == '')`
  - **Unit**: `Unit | Unit_PO | (Internal + MCAAMDEVICEBRANDID != '')`
  - **Vendor**: `Vendor`

### Blind spots / assumptions
- Internal invoice routing hinges on whether `MCAAMDEVICEBRANDID` is blank/non-blank.
- Edits can re-include older invoices through `ModifiedDateTime`.

---

## MSRP (`SalesTable`)
Record is captured when all are true:
- Current company
- `SalesStatus == Invoiced`
- `AMDeviceId != ''`
- `SalesPoolId == 'UNIT'`
- Time condition (important differences below)

### MSRP time-condition differences by probe type
- `quickProbeMsrpExists`: **Created OR Modified** in window.
- `fullProbeMsrpCount`: **Modified only** in window.
- `enumerateMsrpRecIds`: **Modified only** in window.

### Blind spots / gotchas
1. **Your discovered gotcha is real**: if a record is invoiced and then modified later, MSRP net will pick it up (because ModifiedDateTime qualifies).
2. **Probe inconsistency risk**:
   - quick probe may return `true` due to CreatedDateTime-only match,
   - while full/enumerate return 0/empty because they require ModifiedDateTime.
3. No explicit invoice-date constraint exists; inclusion is based on current status + timestamp filters only.
4. Excludes non-UNIT sales pools and records without `AMDeviceId`.

### Direct answer: "If modified date is after invoicing, will MSRP pick it up?"
**Yes.** As long as the row is currently `SalesStatus::Invoiced`, in `SalesPoolId == 'UNIT'`, has non-empty `AMDeviceId`, and `ModifiedDateTime` is inside the scan window, it will be included by MSRP full/enumerate nets.

---

## Certificate of Origin (`SalesTable` + device joins)
Record is captured when all are true:
- Current company
- `SalesStatus == Invoiced`
- `AMDeviceId != ''`
- `SalesPoolId == 'UNIT'`
- Device joins resolve `AMDeviceTable` and `AMDeviceTableMaster`
- `BodyId` in `MHA | MHB | MHC`
- `CreatedDateTime` OR `ModifiedDateTime` inside window

### Blind spots / assumptions
- Missing/broken device joins exclude otherwise qualifying SalesTable rows.
- Body styles outside MHA/MHB/MHC are intentionally invisible.
- Like other nets, later edits can re-surface old sales orders.

---

## Drag-net limitations summary (cross-cutting)
1. **Not event-sourced**: these nets infer business events from current header row timestamps.
2. **Re-capture behavior**: modified records re-enter windows; dedupe/downstream arbitration is required.
3. **Boundary sensitivity**: `_toUtc` is exclusive.
4. **Current-state dependency**: subtype/status/pool/device fields are evaluated as they exist now.
5. **MSRP inconsistency**: quick probe uses Created OR Modified, but full/enumerate use Modified only.

---

## Recommended validation checks before production runs
1. Run a targeted window where `_fromUtc`/`_toUtc` straddle known edge records at exact boundary timestamps.
2. Compare MSRP quick vs full/enumerate results for the same window to surface Created-only records.
3. Validate records modified post-invoice to confirm expected re-capture volume.
4. Spot-check Internal classification splits (Confirmation/Invoice) when control fields change.

