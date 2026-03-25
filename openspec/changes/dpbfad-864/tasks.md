# Internal Tasks - Student Capital Adjustment Fix

## Backend Tasks (bidding-api)

### 1. Update `AdjustmentRequestToEntityMapper.php` Constructor [x]
- Add `StudentCapitalService` and `StudentCreditService` to the `AdjustmentRequestToEntityMapper` constructor in `bidding-api/src/Domain/Adjustment/Mapper/AdjustmentRequestToEntityMapper.php`. [x]
- Ensure they are injected as `private readonly` properties. [x]

### 2. Update `AdjustmentRequestToEntityMapper.php` Logic [x]
- Update the `populate` method to fetch the "true" capital left using `StudentCapitalService::getCapital($student)->getLeft()`. [x]
- Use this true balance for validation when `action == 'remove'`. [x]
- Update `StudentData::setRemainingCapital()` based on the NEW true balance (true balance - adjustment amount). [x]
- Similarly, fetch the "true" credits earned using `StudentCreditService::getCredits($student)->getTaken()`. [x]
- Update `StudentData::setCreditTaken()` accordingly during adjustment (true earned + adjustment amount). [x]

### 3. Verification
- Manually test removing capital from a student who has spare capital but potentially out-of-sync `StudentData::remainingCapital`.
- Verify that `StudentData` is updated with the correct value after the adjustment.

### 4. Regression Testing
- Verify that "Add" adjustments still work correctly and update `StudentData` as expected.
- Verify that "Remove" adjustments still block if the student truly doesn't have enough spare capital (e.g., removing more than the spares).

## Expected Outcomes
- Admin can consistently remove capital when there is "spare" capital visible on the dashboard.
- Manual adjustments act as a "Sync" mechanism for `StudentData` denormalized fields.
