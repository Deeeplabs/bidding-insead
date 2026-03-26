## Context

The PM/PO dashboard shows three header metrics: Active Campaigns, Bidding Status Overview, and Credit Progress. These aggregate data across all active campaigns for the selected programme.

**Current state:**
- Labels were recently updated to include clarifying parenthetical text indicating cross-campaign scope
- The Bidding Status Overview calculation aggregates students across all Pre-Bidding and Bidding rounds — functional
- The Credit Progress calculation has a reported issue: the credit range boundaries (`min_credits_per_student` / `max_credits_per_student`) from the bidding round config may not correctly reflect the Final Enrollment credit range when these values differ between modules
**Files involved:**
- Backend: `bidding-api/src/Domain/Dashboard/PMDashboardStatsService.php` — main calculation service
- Backend: `bidding-api/src/Domain/Dashboard/PMDashboardCacheService.php` — cache layer
- Backend: `bidding-api/src/Domain/Campaign/Campaign/CampaignStatsBatchService.php` — batch data helpers

## Goals / Non-Goals

**Goals:**
- Fix the Credit Progress calculation so it accurately counts "on track" students using the correct credit range per Final Enrollment module
- Ensure both PM and PO users see consistent, accurate metrics via the corrected API response
- Verify Bidding Status Overview aggregation is correct

**Non-Goals:**
- Adding new metrics or KPIs to the dashboard
- Changing the Active Campaigns calculation (already correct)
- Any frontend changes (labels and tooltips are already correct in the frontend; the backend API response label field provides clarifying text)
- Modifying the campaign-level stats in `BoxCampaign` (header already correct)
- Adding new API endpoints or changing response DTO shapes
- BP dashboard changes (separate component, different stats)

## Decisions

### 1. Credit Progress — use Final Enrollment's own credit config, not bidding round's

**Decision:** The `getCreditProgress()` method currently reads `min_credits_per_student` and `max_credits_per_student` from the *bidding round* module's config to determine the acceptable credit range. This is incorrect when the Final Enrollment module has its own credit configuration that may differ. The fix will read the credit range from the Final Enrollment module's own phase config instead, falling back to the bidding round config only if the Final Enrollment config does not specify credit boundaries.

**Rationale:** The Credit Progress metric is specifically about Final Enrollment — the credit range governing "on track" status should come from the Final Enrollment phase configuration, which is the authoritative source for what credit range is acceptable at that stage.

**Alternative considered:** Always use the campaign-level credit config (`minCreditsToFulfill` / `maxCreditsToFulfill`). Rejected because the per-module config allows more granular control per bidding cycle, which is the intended setup.

### 2. Cache invalidation — no changes needed

**Decision:** The existing `PMDashboardCacheService` caches stats per program and registers campaign-to-program mappings. Since we are only fixing the calculation logic (not changing the cache key structure or data shape), cache invalidation will continue to work as-is. After deploying, existing caches will naturally expire and recalculate with corrected logic.

**Rationale:** The cache TTL is short enough that stale data will be replaced within one cycle. No forced flush is needed.

### 3. No migration required

**Decision:** No database schema changes. The fix is purely in PHP service logic.

## Risks / Trade-offs

- **[Calculation change affects live data]** → The corrected Credit Progress numbers may differ from what users have been seeing. Mitigation: this is the desired outcome — the previous numbers were inaccurate.
- **[Cache shows stale data briefly after deploy]** → Users may see old (incorrect) numbers until cache expires. Mitigation: cache TTL is short; alternatively, a cache flush can be done post-deploy if needed.
- **[No dedicated PMDashboardStatsService test]** → There are no existing unit tests for this service. Mitigation: add focused unit tests for the corrected `getCreditProgress()` method as part of implementation.

## Open Questions

- Confirm whether any Final Enrollment phase configs actually define `min_credits_per_student` / `max_credits_per_student` in their module config, or if they consistently rely on the bidding round config. This determines whether the fallback path is exercised in practice.
