## Why

The Programme Manager and Programme Operations dashboard header metrics (Bidding Status Overview and Credit Progress) need clarification and calculation fixes. While the labels were recently updated to indicate cross-campaign aggregation, the Credit Progress calculation still has multiple issues:

1. **FE modules with no enrollments are skipped** — the `getCreditProgress()` method skips Final Enrollment modules entirely when no enrolled bids exist yet, causing the eligible student total to be dramatically undercounted (e.g., 2,340 instead of 6,552 when only 5 of 14 FE modules have enrollments).
2. **Credit range source was wrong** — credit boundaries were read from the bidding round config instead of the Final Enrollment module's own phase config.
3. **Config format mismatch** — the config passed to `getEligibleStudentIds()` used the raw module config structure instead of the properly flattened format, potentially causing incorrect student eligibility calculations.

For Bidding Status Overview, the small discrepancy (~65 students) between expected and actual totals needs verification against actual database data — the code logic appears sound but some phases may have different student eligibility configs.

PM and PO users need accurate, clearly-scoped metrics to understand overall bidding health at a glance.

## What Changes

- **Fix Credit Progress — stop skipping FE modules with no enrollments** in `PMDashboardStatsService::getCreditProgress()` — remove the `if (empty($filteredRows)) continue;` guard that skips the entire FE module (including eligible student counting) when there are no enrolled bids. Students must always be counted in the total regardless of enrollment status.
- **Fix Credit Progress — use FE's own credit config** — ensure the credit range (`min_credits_per_student` / `max_credits_per_student`) from each Final Enrollment module's phase config takes priority, falling back to bidding round config only if FE doesn't define credit boundaries.
- **Fix Credit Progress — flatten config for eligibility service** — pass config to `getEligibleStudentIds()` in the same flattened format as `collectBiddingPhases()` does, ensuring student filters and selection are properly applied.
- **Verify Bidding Status Overview calculation** — confirm that the student count correctly aggregates across all Pre-Bidding and Bidding rounds of all active campaigns.

## Capabilities

### New Capabilities

_(none — this is a fix/clarification of existing dashboard metrics)_

### Modified Capabilities

_(no spec-level requirement changes — the existing dashboard capability's intended behavior is correct; this addresses calculation bugs and label clarity)_

## Impact

- **bidding-api**: `src/Domain/Dashboard/PMDashboardStatsService.php` — fix `getCreditProgress()`: remove filteredRows skip, use FE's own credit config, flatten config for eligibility service; verify `getBiddingStatus()` aggregation
- **bidding-api**: `src/Domain/Dashboard/PMDashboardCacheService.php` — cache invalidation may need review after calculation fix
- **Affected roles**: Programme Manager, Programme Operations (both use the same API endpoint)
- **No migration required** — no schema changes, only backend logic fixes
- **No API response shape changes** — existing DTO fields remain unchanged
- **No frontend changes** — backend labels already include clarifying text in API response
- **Existing tests** in `tests/Unit/Domain/Dashboard/` may need updating for corrected calculation
