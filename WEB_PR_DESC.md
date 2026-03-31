# UI Blocking for Previously Enrolled Courses in Add/Drop

## Summary
Jira: 
- https://insead.atlassian.net/browse/DPBFAD-792

This PR improves the student experience during the Add/Drop phase by proactively blocking the selection of courses that have been previously enrolled. Instead of allowing students to select these courses and receiving a server-side error after submission, the UI now disables these options in the "Add Courses" dropdown with a clear explanation.

## Problem Context

Students were able to search for and select courses they had already enrolled in (either in previous programs or earlier in the current round). Attempting to save or submit these duplicate enrollments resulted in a confusing backend error message.

## What Changed

### 1. `bidding-web/src/features/bidding/utils/validation.util.ts`
- Updated `validateCourseAddition()` to check for `is_enrolled` status and `unavailable_reason`.
- Previously, this only validated credit limits. It now serves as a robust guard for all addition-related business rules.

### 2. `bidding-web/src/features/bidding/hooks/use-add-drop-waitlist-form.tsx`
- Modified `handleAddCourse` to use the updated `validateCourseAddition` logic.
- This ensures that even if a student attempts to add a course (e.g., via keyboard or past UI states), the system prevents it and shows a proactive error notification.

### 3. `bidding-web/src/features/bidding/hooks/use-course-options.ts`
- Leverages existing `is_enrolled` metadata from the API to set the `disabled` state on dropdown options.
- Options now display "Previously Enrolled" as the reason when hovered/disabled.

## Impact
- **Proactive Feedback**: Students immediately see which courses are unavailable and why.
- **Reduced Errors**: Eliminates "Already Enrolled" server-side validation failures by preventing the action at the source.

## Testing / Verification Steps
1. Open the Add/Drop page for an active campaign.
2. Click on the "Add Courses" selector.
3. Search for a course you have already enrolled in.
4. Verify that the course appears as disabled in the list with the title "Previously Enrolled".
5. Verify that attempting to add the course (if somehow possible) results in a toast notification: "Course [Name] has been previously enrolled."
