## 1. Move Course Selection Duplication

- [x] 1.1 In `CampaignDuplicationService::duplicateCampaign()`, remove `$newCampaign->setCourseSelection($sourceCampaign->getCourseSelection())` from the `$duplicateMainConfig` block
- [x] 1.2 In `CampaignDuplicationService::duplicateCampaign()`, remove the `CampaignCourseFilter` duplication loop from the `$duplicateMainConfig` block
- [x] 1.3 Add `courseSelection` JSON duplication and `CampaignCourseFilter` record duplication into the `$duplicateProgrammeCourses` block

## 2. Unit Tests

- [x] 2.1 Add/update PHPUnit test: duplicating with `duplicateProgrammeCourses=false` results in no `courseSelection`, no `CampaignCourseFilter` records, and no `AdjustmentCourse` records on the new campaign
- [x] 2.2 Add/update PHPUnit test: duplicating with `duplicateProgrammeCourses=true` copies `courseSelection`, `CampaignCourseFilter` records, and `AdjustmentCourse` records to the new campaign
- [x] 2.3 Add/update PHPUnit test: duplicating with `duplicateMainConfig=true` and `duplicateProgrammeCourses=false` copies main config fields but no course data

## 3. Verification

- [x] 3.1 Verify existing duplication tests still pass (run `phpunit --filter=Duplication`)
- [x] 3.2 Manual smoke test: duplicate a campaign with "Duplicate programme courses" unchecked and confirm course list is empty
