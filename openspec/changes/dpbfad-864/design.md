# Design - Student Capital Adjustment Fix

## Problem Root Cause
The `AdjustmentRequestToEntityMapper` uses `StudentData::getRemainingCapital()` as the source of truth for validation during "Remove" operations. This field is a denormalized value that can easily become out of sync (e.g., if adjustments are imported directly without updating the entity, or if campaign operations don't correctly maintain it).

The admin dashboard correctly calculates the "Spare" capital using `StudentCapitalService`, which sums initial grants + all existing adjustments - total spent from bids.

## Implementation Details

### Domain - bidding-api (Backend)

#### `AdjustmentRequestToEntityMapper.php`
- Inject `StudentCapitalService` and `StudentCreditService` into the constructor.
- In the `populate` method:
    - Get the current true balances from the services:
        - `capitalLeft = $this->studentCapitalService->getCapital($student)->getLeft();`
        - `creditsTaken = $this->studentCreditService->getCredits($student)->getTaken();`
    - Update the validation for `action == 'remove'`:
        - Use the true `capitalLeft` for the comparison instead of `$studentData->getRemainingCapital()`.
        - Use the true `creditsTaken` for sync? Wait, for credits, "Taken" means "Earned". If we remove credits, we reduce "Earned".
    - Update synchronization with `StudentData`:
        - During ANY adjustment mapping (Add or Remove), set the `StudentData` fields to the NEW calculated values after the adjustment is applied. This ensures that every manual adjustment also performs a "Sync" of the student's data.

### Impacted Files
- `bidding-api/src/Domain/Adjustment/Mapper/AdjustmentRequestToEntityMapper.php`

### Testing Strategy
- Unit test for `AdjustmentRequestToEntityMapper` to verify that "Remove" operations succeed based on calculated spare even if `StudentData` is out of sync.
- Verification that `StudentData` is correctly "synced" after the adjustment is applied.

### Security and Performance
- No new security risks as the admin is still required to have `ROLE_ADMIN`.
- The performance impact is minimal as these services are already optimized and used in student listings.
