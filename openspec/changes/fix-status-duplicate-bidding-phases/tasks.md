## 1. Backend Modifications

- [x] 1.1 Locate `CampaignDuplicateService` (or equivalent method handling Campaign/Module duplication).
  - Ensure `actualStartDate`, `actualEndDate`, and runtime `status` properties are set to null/default respectively when cloning modules and phases.
  - If `$duplicateProgrammeCourses` is false, ensure that `course_filters`, `course_selection`, and `selected_course_ids` are cleared from the duplicated `moduleConfig` JSON for both `CampaignModule` and `CampaignPhaseConfig`.

- [x] 1.2 Ensure `Transformer` classes that serialize module/campaign details for the admin frontend output both the `expected` and `actual` date fields correctly without removing either.

## 2. Frontend (Admin Dashboard) Updates

- [x] 2.1 Locate the Campaign/Module list view in `bidding-admin/src/app/(authenticated)/...` and update the date rendering component to inject `Actual Start–End Date (Expected Start–End Date)` when `actualStartDate` and `actualEndDate` are present.
- [x] 2.2 Validate functionality in Admin Dashboard by copying a campaign, making sure new module dates are purely expected, and test manually starting the module to observe the parenthetical date format.
- [x] 2.3 Validate that duplicating without course lists correctly clears any course-related module filters from the UI.

## 3. Verification

- [x] 3.1 Smoke test the campaign duplicate behaviour on local environment.
- [x] 3.2 Smoke test the student portal to verify that default displays remain unaffected and only present the valid current dates.
