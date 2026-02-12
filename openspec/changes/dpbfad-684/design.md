## Context

Campaign duplication in `CampaignDuplicationService` is controlled by four boolean flags: `duplicateMainConfig`, `duplicateCampaignFlow`, `duplicateProgrammeCourses`, and `duplicateBiddingRounds`. The `duplicateProgrammeCourses` flag currently only guards `AdjustmentCourse` record duplication, but two other course-related data items — `courseSelection` JSON and `CampaignCourseFilter` records — are incorrectly guarded by `duplicateMainConfig`.

When a PM duplicates a campaign with "Duplicate programme courses" unchecked, the copied `courseSelection` and `CampaignCourseFilter` data cause `CampaignCourseService::buildAvailableCourse()` to dynamically rebuild the course list from the filter criteria, making it appear as if courses were duplicated.

Separately, the `campaign_executions` table carries a stale foreign key constraint (`campaign_executions_ibfk_2`) from the original `preset_campaign_executions` table. This FK references `preset_modules(id)` instead of `campaign_modules(id)`. When `Version20251212083834` renamed the table and column and added the correct FK (`fk_campaign_executions_campaign_module`), it did not drop the old auto-named FK. This causes `SQLSTATE[23000]: 1452` integrity constraint violations whenever a `CampaignExecution` is inserted, because the `campaign_module_id` value exists in `campaign_modules` but not in `preset_modules`. On MySQL 5.7, the `INFORMATION_SCHEMA` metadata for this renamed column can be inconsistent, sometimes listing the FK under the old column name (`preset_module_id`).

## Goals / Non-Goals

**Goals:**
- Course selection data (`courseSelection` JSON) is only duplicated when `duplicateProgrammeCourses` is true
- Course filter records (`CampaignCourseFilter`) are only duplicated when `duplicateProgrammeCourses` is true
- `AdjustmentCourse` duplication remains under the `duplicateProgrammeCourses` guard (already correct)
- All other duplication behavior remains unchanged
- Drop the stale FK `campaign_executions_ibfk_2` that references `preset_modules(id)`
- Ensure the correct FK `fk_campaign_executions_campaign_module` referencing `campaign_modules(id)` exists
- Ensure the migration is robust against MySQL 5.7 metadata inconsistencies (checking old and new column names)

**Non-Goals:**
- Changing frontend duplication modal behavior (already sends correct values)
- Changing the `DuplicateCampaignRequest` DTO structure
- Retroactively fixing previously duplicated campaigns
- Changing how `CampaignCourseService::buildAvailableCourse()` works
- Modifying the `CampaignExecution` entity mapping (already correctly references `CampaignModule`)

## Decisions

**Move `courseSelection` and `CampaignCourseFilter` duplication to the `duplicateProgrammeCourses` block**

The `courseSelection` JSON field and `CampaignCourseFilter` records are semantically part of "programme courses" configuration — they define which courses are available in a campaign. Moving them to the `$duplicateProgrammeCourses` guard aligns the code with user intent.

Alternative considered: Adding a separate `duplicateCourseFilters` flag. Rejected because it would require DTO changes, frontend changes, and adds UI complexity for something that logically belongs under "programme courses."

**Create a robust, triple-check migration for MySQL 5.7**

Standard Doctrine `Schema` manipulation or simple `INFORMATION_SCHEMA` queries might miss the stale FK if MySQL 5.7 hasn't fully updated its metadata after the rename. The migration `Version20260211000000` will use a **triple-check strategy**:
1. Check for FKs on `campaign_module_id` referencing `preset_modules` (the active column)
2. Check for FKs on `preset_module_id` referencing `preset_modules` (the old column name, in case of stale metadata)
3. Check for the specific constraint name `campaign_executions_ibfk_2` (safety net)

All found constraint names are collected into a unique set (deduplicated) before generating `DROP FOREIGN KEY` SQL to prevent duplicate drop errors.

Alternative considered: Fixing the original migration `Version20251212083834`. Rejected because re-running migrations in production is risky. A new, robust migration is safer.

## Risks / Trade-offs

- **[Risk] PMs who relied on course filters being copied with main config** → This was unintentional behavior. The toggle label "Duplicate programme courses" clearly communicates that unchecking it means no course data is copied. Low risk.
- **[Risk] Campaigns duplicated with main config but without programme courses will start with no course selection** → This is the correct behavior — PMs must configure courses for the new campaign. Mitigation: None needed, this matches user expectation.
- **[Risk] Dropping `campaign_executions_ibfk_2` on production** → The FK references a table (`preset_modules`) that is unrelated to the current `campaign_module_id` data. Removing it has no functional impact other than fixing the constraint violation. The correct FK already exists. Low risk.
