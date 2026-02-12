## Why

When duplicating a campaign with "Duplicate programme courses" unchecked, the duplicated campaign still shows the original campaign's course list. This happens because `courseSelection` JSON and `CampaignCourseFilter` records are duplicated under the `$duplicateMainConfig` guard instead of `$duplicateProgrammeCourses`, causing `CampaignCourseService::buildAvailableCourse()` to dynamically rebuild the course list from the copied filter criteria.

## What Changes

- Move `courseSelection` JSON duplication from the `$duplicateMainConfig` block to the `$duplicateProgrammeCourses` block in `CampaignDuplicationService`
- Move `CampaignCourseFilter` record duplication from the `$duplicateMainConfig` block to the `$duplicateProgrammeCourses` block
- When `duplicateProgrammeCourses` is false, the new campaign will have no course selection criteria, no course filters, and no adjustment courses — resulting in a clean course list that the PM must configure from scratch

## Capabilities

### New Capabilities

_(none — this is a bug fix within existing capability)_

### Modified Capabilities

- `campaign-duplication`: Course-related data (`courseSelection`, `CampaignCourseFilter`, `AdjustmentCourse`) must all be guarded by the `$duplicateProgrammeCourses` flag, not `$duplicateMainConfig`

## Impact

- **bidding-api**: `CampaignDuplicationService` — reorder duplication logic so course selection JSON and course filter records are under the programme courses guard
- **Entities affected**: `Campaign` (`courseSelection` field), `CampaignCourseFilter`, `AdjustmentCourse`
- **No migration required** — this is a logic-only fix, no schema changes
- **No frontend changes required** — the admin UI already sends the correct `duplicate_programme_courses` flag; the backend just needs to respect it properly
- **Backward compatibility**: Existing duplicated campaigns are unaffected; only future duplications will behave correctly
