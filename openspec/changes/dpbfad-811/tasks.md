## 1. Fix BidValidator Minimum Capital Config Key

- [x] 1.1 In `bidding-api/src/Domain/Campaign/ActiveCampaign/Validator/BidValidator.php`, update `validateCapital()` to use `$config['min_capital_per_student'] ?? $config['min_bids_entire_round'] ?? null` for the minimum capital check (line ~220), replacing the current `$config['min_bids_entire_round'] ?? null`

## 2. Fix DummyDataService Bid Generation Capital Constraints

- [x] 2.1 In `bidding-api/src/Service/DummyDataService.php`, update `preparedDummyBidsForCampaign()` to resolve the effective max capital as `$config['max_capital_per_student'] ?: $config['max_bids_entire_round'] ?: 0` and effective min capital as `$config['min_capital_per_student'] ?: $config['min_bids_entire_round'] ?: 0` at the start of the method (around line ~2268)
- [x] 2.2 Move the minimum capital enforcement out of the per-item loop. After the `foreach ($cp ...)` loop ends, check if `$currentBidPoints < $effectiveMinCapital` and if so, add the shortfall to the last bid in the `$bids` array (capped at `$effectiveMaxCapital`)
- [x] 2.3 Remove the unreliable `$isLastItem` minimum enforcement logic currently inside the loop body (lines ~2362-2377)

## 3. Unit Tests

- [x] 3.1 Add/update PHPUnit test for `BidValidator::validateCapital()` covering: min_capital_per_student set, min_bids_entire_round fallback, both null, and draft bypass — in `bidding-api/tests/Unit/Domain/`
- [x] 3.2 ~~Add/update PHPUnit test for `DummyDataService::preparedDummyBidsForCampaign()`~~ Skipped — DummyDataService has 10+ injected dependencies making unit testing impractical. Covered by manual smoke test (4.1)

## 4. Verification

- [x] 4.1 Manual smoke test: create a campaign with `min_capital_per_student`=200, `max_capital_per_student`=1000, run Generate Bid Data, verify all generated students have total bid points between 200 and 1000
- [x] 4.2 Manual smoke test: attempt student bid submission with total points below minimum via the student portal, verify rejection error message
