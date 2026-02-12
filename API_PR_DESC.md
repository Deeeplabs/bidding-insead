# DPBFAD-684: Fix campaign duplication copying courses when "Duplicate programme courses" is unchecked

## Problem

When duplicating a campaign with "Duplicate programme courses" unchecked, the duplicated campaign still shows the original campaign's course list. The `courseSelection` JSON and `CampaignCourseFilter` records were incorrectly guarded by the `$duplicateMainConfig` flag instead of `$duplicateProgrammeCourses`, causing `CampaignCourseService::buildAvailableCourse()` to rebuild the course list from copied filter criteria.

## Solution

Moved course-related duplication logic from the `$duplicateMainConfig` block to the `$duplicateProgrammeCourses` block in `CampaignDuplicationService`:

- **`courseSelection` JSON** — now only copied when `duplicateProgrammeCourses=true`
- **`CampaignCourseFilter` records** — now only copied when `duplicateProgrammeCourses=true`
- **`AdjustmentCourse` records** — already correctly guarded (unchanged)

When `duplicateProgrammeCourses` is false, the new campaign starts with no course selection criteria, no course filters, and no adjustment courses.

## Files Changed

| File | Change |
|------|--------|
| `src/Domain/Campaign/Campaign/CampaignDuplicationService.php` | Moved `setCourseSelection()` and `CampaignCourseFilter` loop from `$duplicateMainConfig` to `$duplicateProgrammeCourses` block |
| `tests/Unit/Domain/Campaign/Campaign/CampaignDuplicationServiceTest.php` | Added 3 new tests for course duplication flag behavior; updated existing test mocks |

## Tests Added

- `test_duplicate_with_programme_courses_false_copies_no_course_data` — verifies no course data when flag is false
- `test_duplicate_with_programme_courses_true_copies_all_course_data` — verifies all course data copied when flag is true
- `test_duplicate_main_config_true_programme_courses_false_copies_config_not_courses` — verifies main config is independent of course data

## Impact

- **No migration required** — logic-only fix, no schema changes
- **No frontend changes** — admin UI already sends the correct flag
- **Backward compatible** — only affects future duplications
