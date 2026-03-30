## 1. Fix BidValidator Minimum Capital Config Key

- [x] 1.1 In `bidding-api/src/Domain/Campaign/ActiveCampaign/Validator/BidValidator.php`, remove the `$config['min_bids_entire_round']` fallback from `validateCapital()`.

## 2. Fix DummyDataService Bugs

- [x] 2.1 In `bidding-api/src/Service/DummyDataService.php`, remove `$min_bids_entire_round` and `$max_bids_entire_round` fallbacks from `$effectiveMinCapital` and `$effectiveMaxCapital` calculation.
- [x] 2.2 Fix `$bidStatus` mutation by assigning `$effectiveStatus` locally and cloning the argument object scope out of the student loop logic.
- [x] 2.3 Introduce tracking array `$selectedCourseIds` to skip multiple classes assigned to the same Course ID.
- [x] 2.4 Replace the `lastBid` point dump with a proportional distribution `while` loop to spread capital shortfall across all generated bids.

## 3. Fix DummyDataService Validation Function

- [x] 3.1 In `validateCapitalDummyData()` method, replace `min_bids_entire_round` with `min_capital_per_student` for capital validation.

## 4. Verification

- [x] 4.1 Verify that running `Generate Bid Data` yields more realistic dummy distributions that sum reliably to >= 200 without throwing 198-point outliers.
- [x] 4.2 Verify that the `BidValidator` rejects under-funded bid submissions strictly on `min_capital_per_student` without defaulting to bid counts.
