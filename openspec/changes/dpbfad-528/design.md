## Context

The PM/PO dashboard shows three header metrics: Active Campaigns, Bidding Status Overview, and Credit Progress. These aggregate data across all active campaigns for the selected programme.

**Current state:**
- Labels were recently updated to include clarifying parenthetical text indicating cross-campaign scope
- The Bidding Status Overview calculation aggregates students across all Pre-Bidding and Bidding rounds — functional (small discrepancy of ~65 students vs manual calculation needs data-level verification)
- The Credit Progress calculation had three issues:
  1. FE modules with no enrolled bids were entirely skipped (`if (empty($filteredRows)) continue;`), causing the total to be dramatically undercounted (e.g., 2,340 instead of 6,552)
  2. Credit range boundaries were read from the bidding round config instead of the FE module’s own phase config
  3. The config passed to `getEligibleStudentIds()` was in raw module config format instead of the flattened format expected by the eligibility service
**Files involved:**
- Backend: `bidding-api/src/Domain/Dashboard/PMDashboardStatsService.php` — main calculation service
- Backend: `bidding-api/src/Domain/Dashboard/PMDashboardCacheService.php` — cache layer
- Backend: `bidding-api/src/Domain/Campaign/Campaign/CampaignStatsBatchService.php` — batch data helpers

## Goals / Non-Goals

**Goals:**
- Fix the Credit Progress calculation so it accurately counts eligible students even when FE modules have no enrollments yet
- Fix the Credit Progress calculation to use the correct credit range per Final Enrollment module
- Fix config format consistency so eligibility service receives properly flattened config
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

### 1. Credit Progress — always count eligible students regardless of enrollment status

**Decision:** The `getCreditProgress()` method must always count eligible students in the total, even when a Final Enrollment module has no enrolled bids yet. The previous code had `if (empty($filteredRows)) continue;` which skipped the entire FE module (including eligible student counting) when there were no enrollments. This caused the total to be dramatically undercounted — e.g., showing 2,340 instead of 6,552 when only 5 of 14 FE modules had enrollments.

**Rationale:** The total represents "eligible students across all Final Enrollment rounds". A student is eligible based on campaign configuration, not on whether they have enrolled. FE modules in early stages (before students enroll) must still contribute to the total denominator.

### 2. Credit Progress — use Final Enrollment's own credit config, not bidding round's

**Decision:** The `getCreditProgress()` method currently reads `min_credits_per_student` and `max_credits_per_student` from the *bidding round* module's config to determine the acceptable credit range. This is incorrect when the Final Enrollment module has its own credit configuration that may differ. The fix will read the credit range from the Final Enrollment module's own phase config instead, falling back to the bidding round config only if the Final Enrollment config does not specify credit boundaries.

**Rationale:** The Credit Progress metric is specifically about Final Enrollment — the credit range governing "on track" status should come from the Final Enrollment phase configuration, which is the authoritative source for what credit range is acceptable at that stage.

**Alternative considered:** Always use the campaign-level credit config (`minCreditsToFulfill` / `maxCreditsToFulfill`). Rejected because the per-module config allows more granular control per bidding cycle, which is the intended setup.

### 3. Credit Progress — flatten config for eligibility service

**Decision:** The config passed to `getEligibleStudentIds()` in `getCreditProgress()` must be flattened the same way as in `collectBiddingPhases()`: extract `moduleConfig['config']` and merge with `student_selection`. Previously, the raw `getModuleConfig()` object was passed, which has a nested structure (`{"config": {...}, "student_selection": {...}}`), while the eligibility cache key builder expects `student_filters`, `selected_student_ids`, and `student_selection` at the top level.

**Rationale:** Consistency with the config format used by `collectBiddingPhases()` for bidding status, ensuring the same eligibility logic applies. The mismatch could cause incorrect cache keys and potentially different eligible student counts.

### 4. Cache invalidation — no changes needed

**Decision:** The existing `PMDashboardCacheService` caches stats per program and registers campaign-to-program mappings. Since we are only fixing the calculation logic (not changing the cache key structure or data shape), cache invalidation will continue to work as-is. After deploying, existing caches will naturally expire and recalculate with corrected logic.

**Rationale:** The cache TTL is short enough that stale data will be replaced within one cycle. No forced flush is needed.

### 5. No migration required

**Decision:** No database schema changes. The fix is purely in PHP service logic.

## Risks / Trade-offs

- **[Credit Progress total will increase significantly]** → After the fix, the total will jump from ~2,340 to ~6,552 (for the DL INT MBA programme) because all 14 FE modules now contribute eligible students. This is the desired outcome — previous numbers were undercounting.
- **[Calculation change affects live data]** → The corrected Credit Progress numbers may differ from what users have been seeing. Mitigation: this is the desired outcome — the previous numbers were inaccurate.
- **[Cache shows stale data briefly after deploy]** → Users may see old (incorrect) numbers until cache expires. Mitigation: cache TTL is short; alternatively, a cache flush can be done post-deploy if needed.
- **[Config format fix may change eligible counts]** → Flattening the config properly means student filters are now correctly applied in `getCreditProgress()`. If some FE phases had student filters that were previously ignored, the eligible counts may change. This is correct behavior.
- **[Bidding Status small discrepancy]** → A ~65-student difference between expected and actual Bidding Status totals was observed on DL INT MBA. The code logic is sound; this may be caused by a phase with different student eligibility config or an additional phase not accounted for in the manual calculation. Needs data-level verification.

## Open Questions

- ~~Confirm whether any Final Enrollment phase configs actually define `min_credits_per_student` / `max_credits_per_student` in their module config, or if they consistently rely on the bidding round config.~~ **Resolved**: The code now checks FE config first, falls back to bidding round config.
- Verify the ~65-student Bidding Status discrepancy on DL INT MBA by checking actual phase counts and eligible student configs in the database.
