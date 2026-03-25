# Proposal - Student Capital Adjustment Fix

## Why
Admins are currently blocked from removing capital from students via the "Settings: Student List" if the denormalized `StudentData::remainingCapital` field is out of sync with the actual calculated balance. This occurs because `AdjustmentRequestToEntityMapper` performs a strict safety check against this denormalized field, which can be lower than the student's actual "spare" capital (calculated from total grants + adjustments - spent points). PMs see "spare" capital on the dashboard, leading to frustration when actions are unexpectedly blocked.

## What Changes
- Refactor `AdjustmentRequestToEntityMapper` to use `StudentCapitalService` and `StudentCreditService` as the source of truth for validation during adjustment mapping.
- Update `StudentData` fields (`remainingCapital`, `creditTaken`) during the adjustment mapping process to ensure they are synchronized with the newly calculated totals.
- Allow adjustment removal as long as the student has sufficient "true" spare capital/credits calculated from their entire bidding history.

## Capabilities
### New Capabilities
- `student-capital-adjustment-sync`: Use real-time balance calculations for adjustment validation and automatically synchronize denormalized `StudentData` fields during manual adjustments.

## Impact
- **bidding-api**:
  - `AdjustmentRequestToEntityMapper`: Update logic to inject services and use true balances.
  - `StudentDataRepository`: Use for persisting synchronized student data.
- **Entities affected**: `StudentData` (syncing denormalized fields), `Adjustment` (created with correct amount).
- **Migration**: No schema changes required — only behavioral data synchronization.
- **Frontend changes**: None required as the admin dashboard already displays correct labels and statuses.
- **Backward compatibility**: Preserves existing "Add" functionality while making "Remove" more robust.
