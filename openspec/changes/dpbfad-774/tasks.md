## 1. Repository Methods

- [x] 1.1 Add `countActivePromotions(?int $programId = null): int` method to `bidding-api/src/Repository/PromotionRepository.php` — query Promotion entities joined to PromotionStatus where `status.id = PromotionStatus::ACTIVE`, optionally filtered by `program.id`
- [x] 1.2 Add `countActiveStudents(): int` method to `bidding-api/src/Repository/StudentRepository.php` — count Student entities joined to `status` and `promotion.status` where `status.id = StudentStatus::ACTIVE` AND `promotion.status.id = PromotionStatus::ACTIVE` (mirrors `queryPaginated()` filters)

## 2. Dashboard Stats Service

- [x] 2.1 Inject `PromotionRepository` and `StudentRepository` into `DashboardStatsService` constructor (`bidding-api/src/Domain/Dashboard/DashboardStatsService.php`)
- [x] 2.2 Update `getProgrammeGovernanceOverview()` to add `promotions` key using `PromotionRepository::countActivePromotions($programmeId)`. Keep existing `programme_operations` key unchanged.
- [x] 2.3 Update `getUserAccessSummary()` to replace the `students` calculation — call `StudentRepository::countActiveStudents()` instead of `countUsersByRole(ROLE_STUDENT)`
- [x] 2.4 Update `getUserAccessSummary()` to compute `total_users` as the sum of `business_partners + programme_managers + programme_operators + students` instead of a separate COUNT query

## 3. Response DTO

- [x] 3.1 Add `public int $promotions` property to `ProgrammeGovernanceOverview` class in `bidding-api/src/Controller/Api/Dashboard/DashboardStatsResponse.php`
- [x] 3.2 Pass `promotions` from stats array to `ProgrammeGovernanceOverview` constructor in `DashboardController::getBPDashboardStats()` — this was the missing wiring that caused the entire endpoint to crash with a TypeError, resulting in all dashboard values showing 0

## 4. Frontend Update

- [x] 4.1 Update `bidding-admin/src/components/dashboard/dashboard-business-partner.tsx` — change the "Promotion" MetricCard to read `bpStats?.programme_governance_overview?.promotions` instead of `programme_operations`
- [x] 4.2 Update dashboard TypeScript types in `bidding-admin/src/src/dashboard/` to add `promotions` field to the `ProgrammeGovernanceOverview` type (if types exist)

## 5. Unit Tests

- [x] 5.1 Add/update PHPUnit test for `DashboardStatsService::getBPDashboardStats()` in `bidding-api/tests/Unit/Domain/Dashboard/` — verify `promotions` returns active promotion count, `total_users` equals sum of roles, and `students` counts active students only

## 6. Smoke Test

- [ ] 6.1 Manual verification: call `GET /dashboard/bp/stats` and confirm `programme_governance_overview.promotions` is present and `user_access_summary.total_users` equals sum of four role counts
