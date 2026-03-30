## Why

The "Generate Bid Data" function created bids with point distributions confusing to QA, leading to bug DPBFAD-811 where QA reported bid points below the minimum capital (e.g. 200). The true root causes were multiple logical bugs rather than missing minimum enforcement:
- The `$effectiveMinCapital` calculation incorrectly fell back to `min_bids_entire_round` if `min_capital_per_student` was missing, incorrectly mixing a minimum count with minimum points.
- The dummy generator dumped any capital shortfall entirely onto the single last class, resulting in one massive bid and multiple tiny bids (which QA saw as "under 200"). 
- Generating dummy bids for multiple sections of the same course due to lack of de-duplication in the ClassPromotions query.
- Mutating the outer loop's `$bidStatus` to `'draft'` on the first under-credited student, forcing all subsequent students to generate hidden draft bids.

## What Changes

- **Fix `BidValidator.validateCapital()` and DummyDataService Config Reads**: Completely remove the illogical fallback from `min_capital_per_student` to `min_bids_entire_round`.
- **Fix `DummyDataService.preparedDummyBidsForCampaign()`**: 
  - Ensure the `$bidStatus` variable is safely cloned and not mutated across the student loop.
  - Filter `ClassPromotions` to ensure a student doesn't receive dummy bids for different sections of the same course.
  - Distribute any capital shortfall relatively evenly across all placed bids (up to 50 points per bid incrementally) instead of dumping it all onto the last bid.

## Capabilities

### Modified Capabilities
- `bid-capital-validation`: Backend validation ensuring bid submissions (both real and generated) respect the configured min/max capital per student, without incorrect fallbacks to minimum block counts.

## Impact

- **bidding-api only** — no admin or web frontend changes needed.
- `src/Domain/Campaign/ActiveCampaign/Validator/BidValidator.php` — fix `validateCapital()` logic.
- `src/Service/DummyDataService.php` — fix `preparedDummyBidsForCampaign()` loop mutations, course de-duplication, and shortfall distribution.
- No database migration required.
- No API response shape changes.
