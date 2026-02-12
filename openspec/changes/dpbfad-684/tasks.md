## 1. Move Course Selection Duplication

- [x] 1.1 In `CampaignDuplicationService::duplicateCampaign()`, remove `$newCampaign->setCourseSelection($sourceCampaign->getCourseSelection())` from the `$duplicateMainConfig` block
- [x] 1.2 In `CampaignDuplicationService::duplicateCampaign()`, remove the `CampaignCourseFilter` duplication loop from the `$duplicateMainConfig` block
- [x] 1.3 Add `courseSelection` JSON duplication and `CampaignCourseFilter` record duplication into the `$duplicateProgrammeCourses` block

## 2. Unit Tests

- [x] 2.1 Add/update PHPUnit test: duplicating with `duplicateProgrammeCourses=false` results in no `courseSelection`, no `CampaignCourseFilter` records, and no `AdjustmentCourse` records on the new campaign
- [x] 2.2 Add/update PHPUnit test: duplicating with `duplicateProgrammeCourses=true` copies `courseSelection`, `CampaignCourseFilter` records, and `AdjustmentCourse` records to the new campaign
- [x] 2.3 Add/update PHPUnit test: duplicating with `duplicateMainConfig=true` and `duplicateProgrammeCourses=false` copies main config fields but no course data

## 3. Fix Stale Foreign Key on campaign_executions

- [x] 3.1 Create migration `Version20260211000000` to drop stale FK `campaign_executions_ibfk_2` that references `preset_modules(id)` instead of `campaign_modules(id)`
- [x] 3.2 In the migration, implement triple-check logic: check `campaign_module_id`, check old `preset_module_id`, and check constraint name `campaign_executions_ibfk_2`
- [x] 3.3 In the migration, use deduplication map to ensure each found constraint is dropped only once (avoid double-drop error)
- [x] 3.4 In the migration, ensure the correct FK `fk_campaign_executions_campaign_module` referencing `campaign_modules(id)` exists (idempotent)

## 4. Verification

- [x] 4.1 Verify existing duplication tests still pass (run `phpunit --filter=Duplication`)
- [x] 4.2 Manual smoke test: duplicate a campaign with "Duplicate programme courses" unchecked and confirm course list is empty
- [x] 4.3 Verify campaign execution creation no longer throws FK constraint violation after migration
