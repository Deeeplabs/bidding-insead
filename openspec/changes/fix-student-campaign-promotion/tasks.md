## 1. Add JSON-based Cross-Promotion Campaign Discovery

- [x] 1.1 Add `findOpenCampaignsByStudentIncludeInConfig(string $studentId): array` method to an appropriate repository (e.g., `CampaignPhaseConfigRepository` or a new method in `CampaignStudentFilterRepository`). This method queries `campaign_phase_configs.module_config` JSON using LIKE matching for `student_include` entries containing the student ID. It should join through `campaign_executions` → `campaigns` table and filter by `c.status = 'open'`, `c.isDeleted = false`, `c.endDate >= NOW()`, and `pc.isActive = true`, `pc.isDeleted = false`. Return an array of `Campaign` entities.

  **File:** `bidding-api/src/Repository/CampaignPhaseConfigRepository.php`

- [x] 1.2 Update `ActiveCampaignService::getActiveBiddingRounds` (in `bidding-api/src/Domain/Campaign/ActiveCampaign/ActiveCampaignService.php`) to add a third campaign discovery step after the existing cross-promotion lookup (after line 169). Call the new `findOpenCampaignsByStudentIncludeInConfig` method. For each discovered campaign, skip if already in `$existingCampaignIds`, then perform the same `getCurrentActivePhaseDetails` → `isStudentEligibleForCampaign` check as the existing cross-promotion block. Update `$existingCampaignIds` after the new block to prevent duplicates.

  **File:** `bidding-api/src/Domain/Campaign/ActiveCampaign/ActiveCampaignService.php`

- [x] 1.3 Inject the repository dependency into `ActiveCampaignService` constructor if not already available. Ensure the new repository is autowired properly.

  **File:** `bidding-api/src/Domain/Campaign/ActiveCampaign/ActiveCampaignService.php`


## 2. Verification and Testing

- [ ] 2.1 Manual verification: Create a campaign for promotion A with a Pre-bidding module. Add a student from promotion B to the Pre-bidding module config with `student_include` filter (via the admin UI module configuration). Impersonate as the student → verify the campaign appears on the student dashboard.

- [ ] 2.2 Verify phase-specific visibility: With the same setup, verify that when the active phase changes to Bidding Round (where student B is NOT included), the campaign does NOT appear for student B.

- [ ] 2.3 Verify existing behavior: Confirm that students from the same promotion still see campaigns normally. Confirm that campaign-level `student_include` filters (in `campaign_student_filters` table) still work via the existing `findOpenCampaignsByStudentInclude` path.

- [ ] 2.4 Verify de-duplication: If a student is found via both table-based AND JSON-based lookup, the campaign should appear only once.

- [ ] 2.5 Smoke test across all affected scenarios to verify no regression in student dashboard campaign visibility.
