# BP Dashboard Stats Accuracy Fixes

## Problem
The `GET /dashboard/bp/stats` endpoint returns incorrect statistics for the Business Partner Dashboard:

1. **Endpoint crash (all stats show 0)** — The `ProgrammeGovernanceOverview` DTO requires 4 constructor arguments but `DashboardController::getBPDashboardStats()` only passes 3, omitting `promotions`. This causes a PHP `TypeError` at runtime, crashing the entire endpoint and returning an error response — the frontend falls back to displaying **0 for all values**.
2. **Promotion count missing** — The `programme_governance_overview` response had no `promotions` field. The frontend used `programme_operations` (Tier 2 ProgramManager count) as a proxy, which returns the wrong number (e.g. **1** instead of the actual **5+** active promotions for a programme).
3. **`total_users` inflated** — Calculated via `COUNT(*)` on all non-deleted User rows (e.g. **8,227**), while per-role counts use the `user_role` join table. Users with zero or multiple roles cause total ≠ sum of categories (7 + 8 + 2 + 8,217 = **8,234 ≠ 8,227**).
4. **`students` count misaligned** — Counts User entities with `ROLE_STUDENT` regardless of status (e.g. **8,217**), while the Students tab filters `Student` entities by `StudentStatus::ACTIVE` AND `PromotionStatus::ACTIVE`, producing a lower count.

## Goal
Fix the `GET /dashboard/bp/stats` response so that:
- A new `promotions` field returns the count of active Promotion entities (filtered by programme when selected)
- `total_users` equals the sum of `business_partners + programme_managers + programme_operators + students`
- `students` counts active Student entities with active Promotions, matching the Students tab

## Changes Made

1. **`src/Repository/PromotionRepository.php`**
   - Added `countActivePromotions(?int $programId = null): int` — counts Promotion entities joined to PromotionStatus where `status.id = PromotionStatus::ACTIVE`, optionally filtered by `program.id`.

2. **`src/Repository/StudentRepository.php`**
   - Added `countActiveStudents(): int` — counts Student entities where `StudentStatus::ACTIVE` AND `PromotionStatus::ACTIVE`, mirroring the `queryPaginated()` join/filter logic used by the Students tab.

3. **`src/Domain/Dashboard/DashboardStatsService.php`**
   - Injected `PromotionRepository` and `StudentRepository` as new constructor dependencies.
   - `getProgrammeGovernanceOverview()`: added `promotions` key via `PromotionRepository::countActivePromotions($programmeId)`. Existing `programme_operations` key unchanged for backward compatibility.
   - `getUserAccessSummary()`: replaced `students` calculation — calls `StudentRepository::countActiveStudents()` instead of `countUsersByRole(ROLE_STUDENT)`.
   - `getUserAccessSummary()`: removed separate `COUNT(*)` query for `total_users` — now computed as `business_partners + programme_managers + programme_operators + students`.

4. **`src/Controller/Api/Dashboard/DashboardStatsResponse.php`**
   - Added `public int $promotions` property to `ProgrammeGovernanceOverview` DTO class.

5. **`src/Controller/Api/Dashboard/DashboardController.php`**
   - Added `promotions: $stats['programme_governance_overview']['promotions']` to the `ProgrammeGovernanceOverview` constructor call in `getBPDashboardStats()`. This was the missing wiring that caused the entire endpoint to crash with a `TypeError`, resulting in all dashboard values showing 0.

6. **`tests/Unit/Domain/Dashboard/DashboardStatsServiceTest.php`** *(new)*
   - Verifies `promotions` returns the active promotion count from `PromotionRepository`.
   - Verifies `total_users` equals the sum of the four role counts.
   - Verifies `students` uses `StudentRepository::countActiveStudents()`.

## API Response Change

The `programme_governance_overview` object now includes `promotions`:

```json
{
  "programme_governance_overview": {
    "active_campaigns": 84,
    "programme_managers": 8,
    "programme_operations": 1,
    "promotions": 5
  }
}
```

This is an **additive, non-breaking change**. The existing `programme_operations` field is preserved with its original meaning (Tier 2 ProgramManager count).

## Impact
- No migrations required — query logic corrections only, no schema changes.
- No breaking changes — all existing response fields remain unchanged.
- `total_users` will decrease for systems where users without roles or with multiple roles inflated the count.
- `students` will decrease for systems with inactive students or students in inactive promotions.

## Testing / Verification Steps
1. `GET /dashboard/bp/stats` — confirm `programme_governance_overview.promotions` is present and equals the count of active promotions.
2. `GET /dashboard/bp/stats?programme_id={id}` — confirm `promotions` filters to the selected programme.
3. Verify `user_access_summary.total_users` = `business_partners + programme_managers + programme_operators + students`.
4. Verify `user_access_summary.students` matches the total shown on the Students tab.
