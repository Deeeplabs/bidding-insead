# Fix campaign duplication copying courses when "Duplicate programme courses" is unchecked

## Problem

When duplicating a campaign with "Duplicate programme courses" unchecked, the duplicated campaign still shows the original campaign's course list. The `courseSelection` JSON and `CampaignCourseFilter` records were incorrectly guarded by the `$duplicateMainConfig` flag instead of `$duplicateProgrammeCourses`, causing `CampaignCourseService::buildAvailableCourse()` to rebuild the course list from copied filter criteria.

Additionally, a stale foreign key constraint (`campaign_executions_ibfk_2`) on the `campaign_executions` table was still referencing `preset_modules(id)` instead of `campaign_modules(id)`. This caused an integrity constraint violation (`SQLSTATE[23000]: 1452`) whenever a new `CampaignExecution` was created. On MySQL 5.7, metadata for renamed columns can sometimes become inconsistent, making this stale FK hard to detect via standard schema introspection if it's still attached to the old column name (`preset_module_id`) in `INFORMATION_SCHEMA`.

## Solution

### 1. Course Duplication Logic Fix

Moved course-related duplication logic from the `$duplicateMainConfig` block to the `$duplicateProgrammeCourses` block in `CampaignDuplicationService`:

- **`courseSelection` JSON** — now only copied when `duplicateProgrammeCourses=true`
- **`CampaignCourseFilter` records** — now only copied when `duplicateProgrammeCourses=true`
- **`AdjustmentCourse` records** — already correctly guarded (unchanged)

When `duplicateProgrammeCourses` is false, the new campaign starts with no course selection criteria, no course filters, and no adjustment courses.

### 2. Stale Foreign Key Fix (Robust Migration)

The `campaign_executions` table had a stale FK (`campaign_executions_ibfk_2`) left over from a previous table rename. To handle potential MySQL 5.7 metadata inconsistencies, the new migration `Version20260211000000` uses a **robust triple-check strategy**:

1.  Checks for FKs on `campaign_module_id` referencing `preset_modules` (active column)
2.  Checks for FKs on `preset_module_id` referencing `preset_modules` (old column name, covering stale metadata)
3.  Checks for the specific constraint name `campaign_executions_ibfk_2` (safety net)

All found constraints are collected into a unique set (deduplicated) to prevent double-drop errors, before being dropped via `ALTER TABLE ... DROP FOREIGN KEY`. Finally, it ensures the correct `fk_campaign_executions_campaign_module` exists.

## Files Changed

| File | Change |
|------|--------|
| `src/Domain/Campaign/Campaign/CampaignDuplicationService.php` | Moved `setCourseSelection()` and `CampaignCourseFilter` loop from `$duplicateMainConfig` to `$duplicateProgrammeCourses` block |
| `tests/Unit/Domain/Campaign/Campaign/CampaignDuplicationServiceTest.php` | Added 3 new tests for course duplication flag behavior; updated existing test mocks |
| `migrations/Version20260211000000.php` | New robust migration to drop stale FK `campaign_executions_ibfk_2` via triple-check strategy |

## Tests Added

- `test_duplicate_with_programme_courses_false_copies_no_course_data` — verifies no course data when flag is false
- `test_duplicate_with_programme_courses_true_copies_all_course_data` — verifies all course data copied when flag is true
- `test_duplicate_main_config_true_programme_courses_false_copies_config_not_courses` — verifies main config is independent of course data

## Impact

- **Migration required** — `Version20260211000000` must be run to drop the stale FK constraint. It is safe, idempotent, and touches **zero data rows**.
- **No frontend changes** — admin UI already sends the correct flag
- **Backward compatible** — only affects future duplications; the FK fix resolves the integrity constraint error for all campaign execution creation flows (duplication, preset deployment, phase configuration)
