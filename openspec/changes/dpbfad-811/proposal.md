## Why

The "Generate Bid Data" function creates bids with total bid points far below the configured minimum capital per student. In a campaign configured with min=200 / max=1000 capital, a student received only 6 total bid points. Additionally, the real bid submission validator (`BidValidator.validateCapital()`) uses the wrong config key (`min_bids_entire_round`) instead of `min_capital_per_student` for minimum capital enforcement, meaning the minimum check can be silently skipped for real student submissions too.

## What Changes

- **Fix `BidValidator.validateCapital()`**: Use `min_capital_per_student` (with fallback to `min_bids_entire_round`) for the minimum total capital check, consistent with how max uses `max_capital_per_student`.
- **Fix `DummyDataService.preparedDummyBidsForCampaign()`**: Ensure the generated total bid points always respect `min_capital_per_student` and `max_capital_per_student` constraints. Fix the unreliable last-item minimum enforcement — the minimum adjustment must fire after the loop completes on the last bid actually placed, not only when the loop reaches the array's literal last element.
- **No migration, no entity changes, no API shape changes** — this is a logic-only fix in two existing PHP services.

## Capabilities

### New Capabilities
- `bid-capital-validation`: Backend validation ensuring bid submissions (both real and generated) respect the configured min/max capital per student for the entire bidding cycle.

### Modified Capabilities
<!-- No existing specs to modify -->

## Impact

- **bidding-api only** — no admin or web frontend changes needed.
- `src/Domain/Campaign/ActiveCampaign/Validator/BidValidator.php` — fix `validateCapital()` config key lookup.
- `src/Service/DummyDataService.php` — fix `preparedDummyBidsForCampaign()` minimum enforcement logic.
- No database migration required.
- No API response shape changes.
- No webhook contract changes.
- Existing unit tests in `tests/` may need updating if they cover `BidValidator` or `DummyDataService`.
