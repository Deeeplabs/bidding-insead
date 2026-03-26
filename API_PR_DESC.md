# Fix: Dashboard Inaccurate Stats (Header) — Credit Progress Calculation

## Problem
Jira: https://insead.atlassian.net/browse/DPBFAD-528

The PM/PO dashboard header metric **Credit Progress** had three calculation issues:

1. **FE modules with no enrollments were skipped entirely** — the `getCreditProgress()` method had a `if (empty($filteredRows)) continue;` guard that skipped the entire Final Enrollment module (including eligible student counting) when there were no enrolled bids. This caused the total to be dramatically undercounted. Example: on DL INT MBA, only 5 of 14 FE modules had enrollments, showing 2,340 instead of the expected 6,552 (14 × 468 students).

2. **Wrong credit range source** — the method was using the **bidding round** module's phase config (`min_credits_per_student` / `max_credits_per_student`) to determine whether students are "on track" during Final Enrollment. However, Final Enrollment modules can define their **own** credit boundaries in their phase config, which may differ from the bidding round's config.

3. **Config format mismatch for eligibility service** — the config passed to `getEligibleStudentIds()` used the raw `getModuleConfig()` object (nested structure) instead of the properly flattened format used by `collectBiddingPhases()`. This could cause incorrect student filter application and cache key mismatches.

## Goal
- **Always count eligible students** in the total, regardless of whether FE modules have enrolled bids yet.
- Read credit boundaries from the **Final Enrollment module's own phase config** first.
- Fall back to the bidding round config only when the Final Enrollment config does not define credit boundaries.
- **Flatten config properly** for the eligibility service to ensure consistent student filter application.
- Ensure Bidding Status Overview and Credit Progress aggregation across all active campaigns is correct.

## Changes Made

### 1. `bidding-api/src/Domain/Dashboard/PMDashboardStatsService.php`
- **Removed `if (empty($filteredRows)) continue;`** — FE modules with no enrolled bids now still contribute eligible students to the total. Students with no enrollments are counted in the denominator but not as "on track".
- **Fixed config format for `getEligibleStudentIds()`** — now uses `array_merge($moduleConfig['config'] ?? [], ...)` to flatten the config, matching the format used by `collectBiddingPhases()`.
- Updated `getCreditProgress()` to extract `$feConfig` from the Final Enrollment execution's own phase configs before looking at the bidding round config.
- Credit range resolution follows a two-step priority:
  1. **Final Enrollment phase config** — if `min_credits_per_student` / `max_credits_per_student` are defined, use them.
  2. **Bidding round phase config** (fallback) — only used when the FE config does not define one or both boundaries.
- No changes to `getBiddingStatus()` — verified that `CampaignStatsBatchService::collectBiddingPhases()` already correctly collects both `pre_bidding` and `bidding_round` modules.
- No changes to label text — verified all labels are consistent with the spec format.

### 2. `bidding-api/tests/Unit/Domain/Dashboard/PMDashboardStatsServiceTest.php` *(new)*
- Added unit tests covering:
  - **Credit Progress with FE config** — FE's own credit range takes priority over bidding round config.
  - **Credit Progress fallback** — falls back to bidding round config when FE has no credit boundaries.
  - **Credit Progress with no enrollments** — eligible students are still counted in total when FE has no enrolled bids.
  - **Credit Progress with no Final Enrollment** — returns 0/0 when no active campaigns have FE modules.
  - **Credit Progress multi-campaign aggregation** — totals are summed across all active campaigns.
  - **Bidding Status aggregation** — verifies correct aggregation across multiple Pre-Bidding and Bidding rounds.

## Impact
- **No migrations required** — logic-only fix in an existing service method.
- **No API response shape changes** — existing DTO fields (`on_track`, `total`, `percentage`, `label`) remain unchanged.
- **No frontend changes** — backend labels already include aggregation scope text.
- **Affected roles**: Programme Manager, Programme Operations (both use `GET /v2/api/dashboard/pm/stats`).
- **Expected Credit Progress total change**: On DL INT MBA, total should increase from ~2,340 to ~6,552 (14 FE modules × 468 students now all counted).

## Bidding Status Note
A small discrepancy (~65 students) was observed between the expected Bidding Status total (9,413) and the actual (9,478) on DL INT MBA. The code logic is sound — `collectBiddingPhases()` correctly collects all `pre_bidding` and `bidding_round` modules. The discrepancy likely comes from a phase with different student eligibility config or an additional phase not accounted for in the manual calculation. This needs data-level verification.

## Testing / Verification Steps
1. Log in as a **Programme Manager**. Check the dashboard header **Credit Progress** metric — verify the total reflects ALL Final Enrollment modules (e.g., 14 × 468 = 6,552 for DL INT MBA), and the "on track" percentage uses the FE module's credit range.
2. Log in as a **Programme Manager**. Check the **Bidding Status Overview** — verify the total counts aggregate across all Pre-Bidding and Bidding rounds of all active campaigns.
3. Log in as a **Programme Operations** user. Verify the same metrics are displayed (PO shares the PM API endpoint).
4. Run unit tests: `php bin/phpunit tests/Unit/Domain/Dashboard/PMDashboardStatsServiceTest.php`
