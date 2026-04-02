# Fix: Add/Drop Seat Availability Display & Course Discoverability

## Problem
Jira: [DPBFAD-726](https://insead.atlassian.net/browse/DPBFAD-726)

During the Add/Drop phase, students could not easily identify which course sections had open seats. The course selector dropdown showed courses without any seat availability information, and after enrolling in a course, the available courses list showed stale seat counts until the student navigated away and back.

## Changes

### 1. `CourseOptionContent.tsx` — Display Seat Availability Per Section
- Added a seat availability chip next to each course option in the add/drop course selector dropdown.
- Shows `{available_seats}/{total_seats} seats` for each section.
- Sections with available seats display a green-bordered chip; full sections display a red-bordered chip.
- Chip renders conditionally only when `available_seats` and `total_seats` are present in the course data.

### 2. `campaign.mutations.ts` — Invalidate Cache After Enrollment
- After a successful add/drop submission, the `useSubmitAddDrop` mutation now also invalidates:
  - `campaignKeys.addDrop()` — forces the available courses list to refetch with updated seat counts.
  - `campaignKeys.enrollments()` — forces the student's enrollment list to refresh.
- Previously only `detail` and `lists` were invalidated. The available courses query was stale until manual refresh.

### 3. `add-drop.type.ts` — No Changes Needed
- The `AvailableCourseResponse` type already includes `available_seats`, `total_seats`, and `enrollment_count` fields. No modifications required.

### 4. `campaign.service.ts` / `campaign.queries.ts` — No Changes Needed
- The `getAddDropAvailableCourses` service method already accepts arbitrary query parameters via `Record<string, string>`. The new backend `availability` and `sort_by` parameters can be passed directly by callers without service changes.
- Query keys already include params for proper cache keying.

## Impact
- **Immediate Seat Visibility**: Students can see at a glance which sections have open seats before selecting.
- **Fresh Data After Enrollment**: Seat counts and enrollment lists update automatically after submission — no manual refresh needed.
- **No Breaking Changes**: All changes are additive. Existing behavior is preserved when new parameters are not used.

## Verification Steps
1. Navigate to add/drop page during an active campaign.
2. Open the course selector dropdown — verify each course section shows a seat availability chip (e.g., "3/30 seats").
3. Verify full sections show a red chip ("0/30 seats") and available sections show a green chip.
4. Enroll in a course — verify the course list refreshes automatically and the seat count decrements without manual page refresh.
5. Verify the enrollment list updates immediately to include the newly enrolled course.
