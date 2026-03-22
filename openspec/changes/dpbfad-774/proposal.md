## Why

The Business Partner Dashboard displays incorrect statistics across three sections, causing confusion and unreliable reporting for administrators:

1. **"Promotion" card shows wrong data (Programme Governance Overview):** The card reads `programme_governance_overview.programme_operations`, which counts Tier 2 ProgramManagers for the selected programme — not actual Promotion entities. Screenshot shows **Promotion: 1** while the Create Campaign "Select Promotion" dropdown lists 5+ active promotions (FRANCHISE MIM, FRANCHISE MBS, LVMH MIM 2025, DUNLIN 2025, etc.) for the same programme.

2. **"Total Users" doesn't equal the sum of categories (User Access Summary):** The total counts all non-deleted User rows (screenshot: **8,227**) while per-role counts use the `user_role` join table. Sum of displayed categories: 7 + 8 + 2 + 8,217 = **8,234 ≠ 8,227**. Users with zero or multiple roles cause the mismatch. Meanwhile the User Management page lists ~74 admin users — the 8,227 figure is inflated by Student entities counted via their User row.

3. **"Students" count differs from Students tab (User Access Summary):** Dashboard shows **8,217** (User entities with `ROLE_STUDENT`, all statuses) while the Students tab lists fewer records because `StudentRepository::queryPaginated()` filters on `StudentStatus::ACTIVE` AND `PromotionStatus::ACTIVE`. Inactive students and students in inactive promotions inflate the dashboard figure.

## What Changes

- **Fix "Promotion" metric in Programme Governance Overview**: Add a query to count actual Promotion entities (filtered by programme if selected) and return it as a new `promotions` field. Update the frontend to display `promotions` instead of `programme_operations`.
- **Fix "Total Users" calculation in User Access Summary**: Change total users to be the sum of the per-role counts, or alternatively count only users who have at least one role assigned. This ensures the total equals the sum of displayed categories.
- **Fix "Students" count in User Access Summary**: Align the dashboard student count with the Students tab by counting Student entities where `StudentStatus::ACTIVE` and `PromotionStatus::ACTIVE`, matching the `StudentRepository::queryPaginated()` logic.

## Capabilities

### New Capabilities
- `bp-dashboard-stats-accuracy`: Correct calculation logic for all BP Dashboard statistics — Promotion count, Total Users summation, and Students count alignment with the Students tab.

### Modified Capabilities

## Impact

- **bidding-api**: `DashboardStatsService.php` — modify `getUserAccessSummary()` and `getProgrammeGovernanceOverview()` methods. Add dependency on `PromotionRepository` and `StudentRepository`. Response DTO adds `promotions` field to `programme_governance_overview` (additive, non-breaking).
- **bidding-admin**: `dashboard-business-partner.tsx` — update "Promotion" card to read `promotions` from response instead of `programme_operations`.
- **No migrations required** — all changes are query logic corrections, no schema changes.
- **No breaking changes** — existing response fields are preserved; `programme_operations` remains in the response for backward compatibility.
