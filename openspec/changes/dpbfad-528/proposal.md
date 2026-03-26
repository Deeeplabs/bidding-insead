## Why

The Programme Manager and Programme Operations dashboard header metrics (Bidding Status Overview and Credit Progress) need clarification and calculation fixes. While the labels were recently updated to indicate cross-campaign aggregation, the Credit Progress calculation still has issues — students may be incorrectly counted or the credit range boundaries may not be applied correctly per campaign's bidding round configuration. PM and PO users need accurate, clearly-scoped metrics to understand overall bidding health at a glance.

## What Changes

- **Fix Credit Progress calculation** in `PMDashboardStatsService::getCreditProgress()` — ensure the credit range (`min_credits_per_student` / `max_credits_per_student`) from each campaign's bidding round config is correctly applied when determining "on track" students across all active campaigns' Final Enrollment rounds.
- **Verify Bidding Status Overview calculation** — confirm that the student count correctly aggregates across all Pre-Bidding and Bidding rounds of all active campaigns (XX = sum of students × rounds, YY = completed count).

## Capabilities

### New Capabilities

_(none — this is a fix/clarification of existing dashboard metrics)_

### Modified Capabilities

_(no spec-level requirement changes — the existing dashboard capability's intended behavior is correct; this addresses calculation bugs and label clarity)_

## Impact

- **bidding-api**: `src/Domain/Dashboard/PMDashboardStatsService.php` — fix `getCreditProgress()` calculation logic; verify `getBiddingStatus()` aggregation is correct
- **bidding-api**: `src/Domain/Dashboard/PMDashboardCacheService.php` — cache invalidation may need review after calculation fix
- **Affected roles**: Programme Manager, Programme Operations (both use the same API endpoint)
- **No migration required** — no schema changes, only backend logic fixes
- **No API response shape changes** — existing DTO fields remain unchanged
- **No frontend changes** — backend labels already include clarifying text in API response
- **Existing tests** in `tests/Unit/Domain/Dashboard/` may need updating for corrected calculation
