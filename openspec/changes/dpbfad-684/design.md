## Context

Campaign duplication in `CampaignDuplicationService` is controlled by four boolean flags: `duplicateMainConfig`, `duplicateCampaignFlow`, `duplicateProgrammeCourses`, and `duplicateBiddingRounds`. The `duplicateProgrammeCourses` flag currently only guards `AdjustmentCourse` record duplication, but two other course-related data items — `courseSelection` JSON and `CampaignCourseFilter` records — are incorrectly guarded by `duplicateMainConfig`.

When a PM duplicates a campaign with "Duplicate programme courses" unchecked, the copied `courseSelection` and `CampaignCourseFilter` data cause `CampaignCourseService::buildAvailableCourse()` to dynamically rebuild the course list from the filter criteria, making it appear as if courses were duplicated.

## Goals / Non-Goals

**Goals:**
- Course selection data (`courseSelection` JSON) is only duplicated when `duplicateProgrammeCourses` is true
- Course filter records (`CampaignCourseFilter`) are only duplicated when `duplicateProgrammeCourses` is true
- `AdjustmentCourse` duplication remains under the `duplicateProgrammeCourses` guard (already correct)
- All other duplication behavior remains unchanged

**Non-Goals:**
- Changing frontend duplication modal behavior (already sends correct values)
- Changing the `DuplicateCampaignRequest` DTO structure
- Retroactively fixing previously duplicated campaigns
- Changing how `CampaignCourseService::buildAvailableCourse()` works

## Decisions

**Move `courseSelection` and `CampaignCourseFilter` duplication to the `duplicateProgrammeCourses` block**

The `courseSelection` JSON field and `CampaignCourseFilter` records are semantically part of "programme courses" configuration — they define which courses are available in a campaign. Moving them to the `$duplicateProgrammeCourses` guard aligns the code with user intent.

Alternative considered: Adding a separate `duplicateCourseFilters` flag. Rejected because it would require DTO changes, frontend changes, and adds UI complexity for something that logically belongs under "programme courses."

**Keep the fix backend-only**

The frontend already sends `duplicate_programme_courses: false` by default and the UI correctly presents the toggle. No frontend changes needed.

## Risks / Trade-offs

- **[Risk] PMs who relied on course filters being copied with main config** → This was unintentional behavior. The toggle label "Duplicate programme courses" clearly communicates that unchecking it means no course data is copied. Low risk.
- **[Risk] Campaigns duplicated with main config but without programme courses will start with no course selection** → This is the correct behavior — PMs must configure courses for the new campaign. Mitigation: None needed, this matches user expectation.
