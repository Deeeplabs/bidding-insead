# Fix Duplicate Campaign Phase Module Status and Dates

## Problem

When a Programme Manager (PM) duplicates a campaign, the new campaign's modules inherited stale data from the original campaign. Specifically, `actual_start_date`, `actual_end_date`, and runtime `status` were copied into the new Campaign's phase configurations. This led to incorrect displays, such as showing "filter" copies from the former campaign and preventing the PM from correctly tracking the new campaign's lifecycle. 

Furthermore, if a PM chose to manually start or end a module earlier than scheduled, the system would overwrite the `expected` dates rather than isolating the `actual` dates.

Lastly, when duplicating a campaign and choosing *not* to duplicate the course list, the old module-level course configurations (like `course_filters`, `course_selection`, and `selected_course_ids`) were incorrectly preserved in the duplicated module config JSON, causing stale filter settings to appear on the new campaign.

## Solution

Reworked the duplication process to explicitly sanitize config data, ensuring that expected scheduling properties remain intact while runtime properties are wiped clean. When the `duplicateProgrammeCourses` flag is false, course-specific module configurations are explicitly cleared out. Updated the backend phase models and dashboard response DTOs to map `expected` and `actual` date variables securely.

### Changes Made

**Modified Files:**

1. **`src/Domain/Campaign/Campaign/CampaignDuplicationService.php`**
   - Modified the duplication logic to unset `actual_start_date`, `actual_end_date`, `expected_start_date`, `expected_end_date`, and `status` when cloning `CampaignModule` and `CampaignPhaseConfig`.
   - Implemented logic to clear `course_filters`, `course_selection`, and `selected_course_ids` from the module/phase configurations when the course list is not duplicated (`$duplicateProgrammeCourses` is false).

2. **`src/Domain/Campaign/Campaign/CampaignPhaseService.php`**
   - Updated the `activatePhase` method to preserve the existing `start_date` and `end_date` as `expected_start_date` and `expected_end_date` automatically if missing.
   - Pushed the actual clock start/end values into decoupled `actual_start_date` and `actual_end_date` attributes for both `CampaignPhaseConfig` and root `CampaignModule`.

3. **`src/Domain/Dashboard/Campaign/AdminActivePhaseDto.php`**
   - Expanded DTO to include `expected_start_date`, `expected_end_date`, `actual_start_date`, and `actual_end_date` explicitly.

4. **`src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php`**
   - Modified mapping in Phase/Dashboard building methods to pipe the `actual` and `expected` variables from config into the frontend DTO payload seamlessly.

## Impact

- **Data Consistency:** Duplicating campaigns resets module status purely to schedule configuration, wiping away previous campaign's execution footprints. Modules duplicated without course lists now start completely clean without stale filter states.
- **Accurate PM Tracking:** PMs manually overriding campaign schedules now securely generate an isolated audit of their runtime action without erasing the original plan.
- **Compatibility:** Backwards consistent. Non-duplicate and non-overridden schedules display smoothly out-of-the-box.

## Testing

- [x] Tested duplication engine against live modules: Dates reset cleanly for clones natively.
- [x] Tested manual phase starts: "Actual Start–End Date (Expected Start–End Date)" schema accurately propagates through JSON configurations.
