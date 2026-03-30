## Context

The bidding system allows admins to configure a campaign's bidding round with minimum and maximum capital (total bid points) per student. The UI field "Minimum total bid points (capital) required per student for the entire bidding cycle" maps to config key `min_capital_per_student`, and the maximum maps to `max_capital_per_student`.

Two bugs exist:

1. **BidValidator.validateCapital()** checks `$config['min_bids_entire_round']` for minimum capital instead of `$config['min_capital_per_student']`. Since the admin UI writes to `min_capital_per_student`, the minimum check silently passes when `min_bids_entire_round` is null — meaning students can submit bids with total points below the configured minimum.

2. **DummyDataService.preparedDummyBidsForCampaign()** generates random bid points starting at 1 with `mt_rand(1, ...)`. The loop-end minimum adjustment only fires when the iterator hits `array_key_last($cp)`, but the loop frequently breaks early (due to capital/credit limits), so the adjustment never fires. Additionally, it calls `persistDummyBidsWithoutValidation()`, bypassing all guards.

## Goals / Non-Goals

**Goals:**
- Ensure `BidValidator.validateCapital()` correctly reads `min_capital_per_student` for the minimum total capital check (with fallback to `min_bids_entire_round` for backward compatibility)
- Ensure `DummyDataService.preparedDummyBidsForCampaign()` generates total bid points within the configured `min_capital_per_student` / `max_capital_per_student` range

**Non-Goals:**
- Changing the admin UI or config DTO structure
- Adding per-course bid point minimums (currently disabled and out of scope)
- Modifying the real bid submission endpoint contract or response shape
- Changing entity or database schema

## Decisions

**Decision 1: Fix config key in BidValidator.validateCapital()**

Use `$config['min_capital_per_student'] ?? $config['min_bids_entire_round'] ?? null` for the minimum capital check. This mirrors the max capital check which already uses `$config['max_capital_per_student']`. The fallback to `min_bids_entire_round` preserves backward compatibility for any campaign configs that use that key.

*Alternative considered*: Only use `min_capital_per_student` without fallback — rejected because some historical campaign configs might only have `min_bids_entire_round`.

**Decision 2: Fix DummyDataService minimum enforcement by applying adjustment after the loop**

Move the minimum capital adjustment out of the per-item loop body and apply it after the loop ends, on the last bid actually placed (not the last element in the array). If the cumulative bid points are below `min_capital_per_student` after generating all bids, add the difference to the last bid item. Also use `max_capital_per_student` (with fallback to `max_bids_entire_round`) as the cap for random generation to handle cases where `max_bids_entire_round` is 0 but `max_capital_per_student` is set.

*Alternative considered*: Run BidValidator on dummy data too — rejected because `persistDummyBidsWithoutValidation` intentionally skips validators for other rules (deadline, seat availability, etc.) that don't apply to test data. The fix should be in the generation logic itself.

## Risks / Trade-offs

- **[Risk] Existing campaigns with `min_bids_entire_round` set but `min_capital_per_student` null** → Mitigated by fallback chain: `min_capital_per_student ?? min_bids_entire_round`
- **[Risk] Dummy data total might slightly exceed `max_capital_per_student` after minimum adjustment** → Mitigated by capping the adjusted value at `max_capital_per_student`
- **[Risk] Stricter validation may reject bids that previously went through** → This is intentional and correct behavior; bids violating configured minimums should be rejected
