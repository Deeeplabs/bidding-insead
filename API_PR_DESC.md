# Fix: Student Capital Adjustment

## Problem
Jira: https://insead.atlassian.net/browse/DPBFAD-864

Admins were unable to remove capital or credits from students in the "Settings: Student List" if the denormalized `StudentData::remainingCapital` or `StudentData::creditTaken` fields were out of sync with the actual calculated balance. This occurred because `AdjustmentRequestToEntityMapper` performed strict safety checks against these denormalized fields, which could be lower than the student's actual "spare" capital (calculated from total grants + adjustments - spent points). This led to valid "Remove" operations being blocked despite "spare" capital being visible on the dashboard.

## Goal
- Decouple adjustment validation from denormalized `StudentData` fields.
- Use `StudentCapitalService` and `StudentCreditService` as the source of truth for validation.
- Automatically synchronize `StudentData` denormalized fields during every manual adjustment.

## Changes Made

### 1. `bidding-api/src/Domain/Adjustment/Mapper/AdjustmentRequestToEntityMapper.php`
- Injected `StudentCapitalService` and `StudentCreditService` into the mapper.
- Refactored the `populate` method to fetch the "true" capital left and credits earned using the injected services.
- Updated validation logic for "Remove" actions to use these true balances instead of denormalized fields.
- Updated the mapping logic to automatically update `StudentData::setRemainingCapital()` and `StudentData::setCreditTaken()` with the newly calculated totals (true balance +/- adjustment amount) after every adjustment.
- Ensured `StudentDataRepository::add()` is called to persist the synchronized state.

## Impact
- **No migrations required** — Fixes behavior of existing logic and ensures data consistency through synchronization.
- **Improved Data Integrity** — Manual adjustments now act as a "Sync" mechanism for `StudentData` fields, resolving stale metadata issues.
- **Improved Admin UX** — Prevents valid capital removal operations from being blocked due to out-of-sync local data.

## Testing / Verification Steps
1. Navigate to **"Settings: Student List"** in the Admin Dashboard.
2. Identify a student with "Spare" capital but whose `remainingCapital` field is potentially out of sync (e.g., lower than the amount to remove).
3. Attempt to **"Remove"** capital. Verify the operation succeeds.
4. Verify that the student's `remainingCapital` field is correctly synchronized with the new calculated balance in the database.
5. Attempt to **"Remove"** more capital than the spare amount; verify it is correctly blocked with the error message: *"Cannot reduce more than the balance, cannot be negative."*
6. Repeat for **"Credits"** to verify parity.
