# Spec - Student Capital Adjustment Fix

## Problem Description
Admins cannot remove capital from students in the "Settings: Student List" if `StudentData::remainingCapital` is out of sync or lower than the intended amount to remove, even when the student has enough "spare" capital calculated from their total adjustments and initial grants.

## Root Cause
The `AdjustmentRequestToEntityMapper` performs a safety check against `StudentData::remainingCapital`. However, this field is denormalized and can become out of sync (e.g., after bulk imports or certain campaign operations that don't update it). The admin dashboard shows the calculated "Spare" capital from `StudentCapitalService`, leading to confusion when an action is blocked despite "spare" capital being visible.

## Proposed Changes
1.  **Backend (bidding-api)**:
    -   Modify `AdjustmentRequestToEntityMapper` to use `StudentCapitalService` and `StudentCreditService` to get the actual current balances for validation.
    -   Update `StudentData` fields during the mapping process to ensure they are synchronized with the calculated source of truth.
    -   Ensure that "Remove" operations are allowed as long as the student has enough calculated capital/credits left.

## Scenarios

### Scenario 1: Remove Capital with out-of-sync StudentData
**Given** a student has:
- Initial Capital: 100
- Spent Capital: 20
- Calculated Spare Capital: 80
- `StudentData::remainingCapital`: 10 (incorrectly out of sync)
**When** an admin tries to remove 30 capital
**Then** the operation should **Succeed** (because 80 > 30)
**And** `StudentData::remainingCapital` should be updated to 50 (80 - 30)
**And** an Adjustment record of -30 should be created.

### Scenario 2: Remove Capital exceeding Spare
**Given** a student has:
- Calculated Spare Capital: 20
**When** an admin tries to remove 30 capital
**Then** the operation should **Fail** with message "Cannot reduce more than the balance, cannot be negative."

## Acceptance Criteria
- Admin can remove capital/credits based on the calculated spare balance shown on the dashboard.
- `StudentData` is automatically synchronized during manual adjustments.
