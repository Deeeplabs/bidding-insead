# UI Blocking for Previously Enrolled Courses and Validation Refinement

## Summary
Jira: 
- https://insead.atlassian.net/browse/DPBFAD-792

This PR improves the student experience during both Bidding and Add/Drop phases by proactively blocking the selection of courses that have been previously enrolled or are otherwise unavailable. It also refines the frontend validation logic to ensure consistency and type safety.

## Problem Context

1. **Duplicate Enrollments**: Students were able to select courses they had already enrolled in (either in previous programs or earlier in the current round), leading to server-side errors.
2. **Inconsistent Validation**: Validation logic was scattered and sometimes bypassed the full course context, relying only on credit numbers.
3. **Type Inconsistency**: A mismatch in the `use-bidding-form.ts` hook led to potential runtime or build issues when passing course data to validators.

## What Changed

### 1. `bidding-web/src/features/bidding/utils/validation.util.ts`
- **Object-Based Validation**: Updated `validateCourseAddition()` to accept the full `AvailableCourse` object instead of just a credit number.
- **Enhanced Guards**: Added checks for `is_enrolled` status and `unavailable_reason`. It now blocks courses with a clear message: `"Course {name} has been previously enrolled."`

### 2. `bidding-web/src/features/bidding/hooks/use-bidding-form.ts` & `use-add-drop-waitlist-form.tsx`
- **Consolidated Validation**: Updated `handleAddCourse` in both hooks to pass the full course object to `validateCourseAddition`.
- **Bug Fix**: Resolved a type error in `use-bidding-form.ts` where `number` was being passed instead of an object, ensuring the build process is clean and type-safe.

### 3. `bidding-web/src/features/bidding/hooks/use-course-options.ts`
- **UI State**: Leverages `is_enrolled` metadata to disable dropdown options.
- **User Feedback**: Options now display "Previously Enrolled" as the reason when hovered/disabled.

### 4. `bidding-web/src/features/bidding/utils/__tests__/validation.test.ts`
- **Test Coverage**: Updated unit tests to reflect the new object-based signature and added specific test cases for enrollment and unavailability blocking.

## Impact
- **Proactive Feedback**: Students immediately see which courses are unavailable and why across all bidding-related forms.
- **Consistency**: Centralized business rules for course addition ensuring uniform behavior in Bidding and Add/Drop.
- **Stability**: Fixed type mismatches that were causing build failures.

## Testing / Verification Steps
1. Open the Bidding or Add/Drop page.
2. Search for a course you have already enrolled in.
3. Verify the course is disabled with "Previously Enrolled" text.
4. Attempt to add an unavailable course via alternate means and verify the toast notification: "Course [Name] has been previously enrolled."
5. Run `npm run test` to verify all validation logic passes.
