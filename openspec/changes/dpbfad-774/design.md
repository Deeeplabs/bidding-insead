## Context

The BP Dashboard at `bidding-admin/src/components/dashboard/dashboard-business-partner.tsx` renders three stat sections by calling `GET /dashboard/bp/stats?programme_id={id}`, served by `DashboardController::getBPDashboardStats()` which delegates to `DashboardStatsService`. Three calculations produce incorrect results:

1. **"Promotion" card** displays `programme_governance_overview.programme_operations` — a count of Tier 2 ProgramManagers for the selected programme — instead of actual Promotion entities. Observed: dashboard shows **1** while 5+ promotions exist for the same programme (visible in the Create Campaign "Select Promotion" dropdown: FRANCHISE MIM, FRANCHISE MBS, LVMH MIM 2025, DUNLIN 2025, etc.).
2. **"Total Users"** counts all non-deleted User rows (observed: **8,227**), while per-role counts use the `user_role` join table. Sum of displayed categories: 7 + 8 + 2 + 8,217 = **8,234 ≠ 8,227**. Users with zero or multiple roles break the sum. The User Management page lists ~74 admin users — the inflated figure comes from Student entities counted via their User row.
3. **"Students"** counts User entities with `ROLE_STUDENT` (all statuses, observed: **8,217**), while the Students tab uses `StudentRepository::queryPaginated()` which filters on `StudentStatus::ACTIVE` and `PromotionStatus::ACTIVE`, producing a lower count.

## Goals / Non-Goals

**Goals:**
- Promotion card shows the count of Promotion entities (filtered by programme when selected)
- Total Users equals the sum of the four displayed role categories
- Students count aligns with the Students tab (active students with active promotions)
- Preserve existing API response fields for backward compatibility

**Non-Goals:**
- Redesigning the dashboard layout or adding new stat sections
- Changing the Students tab query logic
- Removing the `programme_operations` field from the response (kept for backward compatibility)

## Decisions

### 1. Add `promotions` field to Programme Governance Overview response

**Decision**: Add a new `promotions` integer field to `ProgrammeGovernanceOverview` DTO. Count Promotion entities where `status = ACTIVE`, optionally filtered by `program_id`. Keep `programme_operations` in the response unchanged.

**Rationale**: Additive-only API change avoids breakage. The frontend switches to read `promotions` instead of `programme_operations` for the "Promotion" card.

**Alternative considered**: Repurpose `programme_operations` to return promotions instead. Rejected because it could break any other consumers relying on the current meaning.

### 2. Compute Total Users as sum of per-role counts

**Decision**: Change `total_users` to equal `business_partners + programme_managers + programme_operators + students` rather than a separate COUNT(*) query. This is computed in PHP, not SQL.

**Rationale**: The dashboard displays five numbers (total + four categories) and users expect total = sum of parts. Users with no role are not actionable from the dashboard perspective. Users with multiple roles already get counted once per role via `COUNT(DISTINCT u.id)`.

**Alternative considered**: Add separate "Unassigned Users" category. Rejected — adds UI complexity without clear business value; can be revisited later.

### 3. Count active students from Student entity, not User entity

**Decision**: Replace the current `countUsersByRole(ROLE_STUDENT)` call with a query against the Student entity: `COUNT(s.id) WHERE s.status = ACTIVE AND s.promotion.status = ACTIVE`. Use `StudentRepository` or inline the query in `DashboardStatsService`. This mirrors `StudentRepository::queryPaginated()`.

**Rationale**: Aligns the dashboard number with what users see in the Students tab. The source of truth for "students" is the Student entity with active status, not the User entity's role assignment.

### 4. Inject PromotionRepository into DashboardStatsService

**Decision**: Add `PromotionRepository` as a constructor dependency to `DashboardStatsService`. Use `PromotionRepository` to count active promotions (possibly via a new `countActivePromotions(?int $programId)` method). Also inject `StudentRepository` for the active student count (or add a `countActiveStudents()` method).

**Rationale**: Follows existing constructor-injection pattern. Service already depends on multiple repositories. New counting methods keep repository logic where it belongs.

## Risks / Trade-offs

- **[Risk] Total Users no longer counts all users in the system** → Acceptable trade-off. The dashboard is a summary view; `total_users` representing the sum of displayed categories is more useful and mathematically consistent. An "all users" count is available through the User Management page.
- **[Risk] Students count changes may surprise users** → Mitigation: the new count matches what the Students tab shows, so it's actually more consistent.
- **[Risk] `programme_operations` field still returned in API** → Acceptable. Field remains for backward compatibility. Frontend simply stops reading it for the "Promotion" card.

## Files to Modify

### Backend (bidding-api)
- `src/Domain/Dashboard/DashboardStatsService.php` — fix all three calculations
- `src/Controller/Api/Dashboard/DashboardStatsResponse.php` — add `promotions` to `ProgrammeGovernanceOverview`
- `src/Repository/PromotionRepository.php` — add `countActivePromotions(?int $programId)` method
- `src/Repository/StudentRepository.php` — add `countActiveStudents()` method

### Frontend (bidding-admin)
- `src/components/dashboard/dashboard-business-partner.tsx` — read `promotions` instead of `programme_operations`
- `src/src/dashboard/` — update TypeScript types if applicable

### Tests
- `tests/Unit/Domain/Dashboard/` — update/add unit tests for `DashboardStatsService`
