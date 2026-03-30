# Fix: Bid Points Capital Validation

## Problem
Jira: https://insead.atlassian.net/browse/DPBFAD-811

The "Generate Bid Data" function produces bids with total bid points far below the configured minimum capital per student. In a campaign configured with min=200 / max=1000 capital, a student received only 6 total bid points. Two root causes were identified:

1. **`BidValidator.validateCapital()` uses the wrong config key** — it reads `min_bids_entire_round` for the minimum capital check instead of `min_capital_per_student`, which is the key the admin UI writes to. This means the minimum capital validation is silently skipped for real student bid submissions when only `min_capital_per_student` is configured.

2. **`DummyDataService.preparedDummyBidsForCampaign()` has unreliable minimum enforcement** — bid points are generated with `mt_rand(1, ...)` (no minimum floor), and the last-item minimum adjustment only fires when the loop reaches `array_key_last($cp)`. Since the loop breaks early due to capital/credit limits, the adjustment never fires for most students. Additionally, `persistDummyBidsWithoutValidation()` bypasses all guards.

## Goal
Ensure bid submissions (both real student bids and admin-generated dummy data) respect the configured `min_capital_per_student` and `max_capital_per_student` constraints from the PM campaign bidding round configuration.

## Changes Made (bidding-api)

1. **`src/Domain/Campaign/ActiveCampaign/Validator/BidValidator.php`**
   - Fixed `validateCapital()`: now reads `min_capital_per_student` with fallback to `min_bids_entire_round` for the minimum total capital check, consistent with how the max check already uses `max_capital_per_student`. This ensures real student bid submissions are properly validated against the configured minimum.

2. **`src/Service/DummyDataService.php`**
   - Introduced `$effectiveMaxCapital` and `$effectiveMinCapital` resolved from `max_capital_per_student ?: max_bids_entire_round` and `min_capital_per_student ?: min_bids_entire_round` respectively. All loop logic now uses these effective values.
   - Replaced the unreliable in-loop `$isLastItem` minimum enforcement with a post-loop adjustment: after all bids are generated, if total points are below `$effectiveMinCapital`, the shortfall is added to the last placed bid (capped at `$effectiveMaxCapital`).
   - The credit minimum check is also moved out of the per-item loop body for clarity.

3. **`tests/Unit/Domain/Campaign/ActiveCampaign/Validator/BidValidatorCapitalTest.php`** (new)
   - 6 test cases covering: rejection below `min_capital_per_student`, acceptance at/above minimum, fallback to `min_bids_entire_round`, no validation when both keys are null, draft bypass, and max capital rejection.

## Impact
- Bid validation now correctly enforces the admin-configured minimum capital (`min_capital_per_student`) for real student bid submissions.
- Generated dummy bid data will always have total bid points within the configured min/max capital range.
- No database migration, entity changes, or API response shape changes.
- No frontend changes required.

## Testing / Verification Steps
- 6 PHPUnit tests pass for `BidValidatorCapitalTest`.
- Manual: Create a campaign with `min_capital_per_student`=200, `max_capital_per_student`=1000, run Generate Bid Data, verify all students have total bid points between 200 and 1000.
- Manual: Submit student bids with total points below the configured minimum, verify rejection error.
