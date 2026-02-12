## Why

When duplicating a campaign with "Duplicate programme courses" unchecked, the duplicated campaign still shows the original campaign's course list. This happens because `courseSelection` JSON and `CampaignCourseFilter` records are duplicated under the `$duplicateMainConfig` guard instead of `$duplicateProgrammeCourses`, causing `CampaignCourseService::buildAvailableCourse()` to dynamically rebuild the course list from the copied filter criteria.

Additionally, creating any new `CampaignExecution` (via duplication, preset deployment, or phase configuration) fails with `SQLSTATE[23000]: Integrity constraint violation: 1452` because a stale foreign key (`campaign_executions_ibfk_2`) on the `campaign_executions` table still references `preset_modules(id)` instead of `campaign_modules(id)`. This FK was left over from the table rename in `Version20251212083834`. On MySQL 5.7, metadata for renamed columns can sometimes become inconsistent, making the FK hard to detect via standard schema introspection.

## What Changes

- Move `courseSelection` JSON duplication from the `$duplicateMainConfig` block to the `$duplicateProgrammeCourses` block in `CampaignDuplicationService`
- Move `CampaignCourseFilter` record duplication from the `$duplicateMainConfig` block to the `$duplicateProgrammeCourses` block
- When `duplicateProgrammeCourses` is false, the new campaign will have no course selection criteria, no course filters, and no adjustment courses — resulting in a clean course list that the PM must configure from scratch
- Drop the stale foreign key `campaign_executions_ibfk_2` that references `preset_modules(id)` using a robust, triple-check strategy to handle potential MySQL 5.7 metadata inconsistencies (checking both new and old column names). Ensure only the correct FK (`fk_campaign_executions_campaign_module` → `campaign_modules(id)`) exists.

## Capabilities

### New Capabilities

_(none — this is a bug fix within existing capability)_

### Modified Capabilities

- `campaign-duplication`: Course-related data (`courseSelection`, `CampaignCourseFilter`, `AdjustmentCourse`) must all be guarded by the `$duplicateProgrammeCourses` flag, not `$duplicateMainConfig`
- `campaign-executions`: The `campaign_executions` table FK on `campaign_module_id` must reference `campaign_modules(id)`, not the legacy `preset_modules(id)`

## Impact

- **bidding-api**: `CampaignDuplicationService` — reorder duplication logic so course selection JSON and course filter records are under the programme courses guard
- **Entities affected**: `Campaign` (`courseSelection` field), `CampaignCourseFilter`, `AdjustmentCourse`, `CampaignExecution` (FK constraint)
- **Migration required** — `Version20260211000000` drops the stale FK `campaign_executions_ibfk_2` and ensures the correct FK exists
- **No frontend changes required** — the admin UI already sends the correct `duplicate_programme_courses` flag; the backend just needs to respect it properly
- **Backward compatibility**: Existing duplicated campaigns are unaffected; only future duplications will behave correctly. The FK fix resolves integrity constraint violations for all campaign execution creation flows.
