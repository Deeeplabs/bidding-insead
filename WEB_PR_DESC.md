# UI Blocking for Previously Enrolled Courses in Add/Drop & Validation Refinement

## Summary
Jira: 
- https://insead.atlassian.net/browse/DPBFAD-792
- https://insead.atlassian.net/browse/DPBFAD-859

This PR improves the student experience during the Add/Drop & Waitlist phase by proactively **disabling** courses that have been previously enrolled in the course selection dropdown. It also refines the frontend validation logic to ensure consistency, type safety, and correct priority ordering of unavailability reasons.

## Problem Context

1. **Enrolled Courses Not Disabled in Dropdown**: In the Add/Drop & Waitlist phase, students were able to select courses they had already enrolled in from the dropdown. The system only blocked them after submission/save with a hard error ("Already enrolled in [Course]"). Enrolled courses should be preemptively disabled at the UI level.
2. **Inconsistent Validation**: Validation logic was scattered and sometimes bypassed the full course context, relying only on credit numbers.
3. **Type Inconsistency**: A mismatch in the `use-bidding-form.ts` hook led to potential runtime or build issues when passing course data to validators.
4. **Incorrect Priority of Enrolled Check**: The `is_enrolled` check in `getUnavailableReason()` was evaluated after credit/deadline/conflict checks, meaning enrolled courses could show misleading reasons instead of "Previously Enrolled".

## What Changed

### 1. `bidding-web/src/features/bidding/utils/validation.util.ts`
- **Object-Based Validation**: Updated `validateCourseAddition()` to accept the full `AvailableCourse` object instead of just a credit number.
- **Enhanced Guards**: Added checks for `is_enrolled` status and `unavailable_reason`. It now blocks courses with a clear message: `"Course {name} has been previously enrolled."`
- **`getUnavailableReason` Priority Fix**: Moved the `is_enrolled` check **above** `exceedsCredit`, `is_passed_deadline`, and `conflictMessage` so enrolled courses are always disabled with "Previously Enrolled" regardless of other conditions.
- **`disabled_reason` Mapping**: Added mapping for `disabled_reason` values sent by the Add/Drop API to display human-readable blocking messages (like "Already Enrolled" and "Already Waitlisted") when cross-module course duplication is detected.

### 2. `bidding-web/src/features/bidding/hooks/use-bidding-form.ts` & `use-add-drop-waitlist-form.tsx`
- **Consolidated Validation**: Updated `handleAddCourse` in both hooks to pass the full course object to `validateCourseAddition`.
- **Bug Fix**: Resolved a type error in `use-bidding-form.ts` where `number` was being passed instead of an object, ensuring the build process is clean and type-safe.
- **Toast Feedback**: Both hooks show toast error notifications if a student tries to add an invalid course (including previously enrolled).

### 3. `bidding-web/src/features/bidding/hooks/use-course-options.ts`
- **Dropdown Disabling**: Leverages `is_enrolled` metadata to disable dropdown options in both regular and backup modes.
- **User Feedback**: Options now display "Previously Enrolled" as the disabled reason.
- **Add/Drop Phase**: Enrolled courses are disabled at the UI level — students cannot select them from the dropdown at all.
- **Bidding Phase**: The bidding form dropdown does **NOT** disable courses bid on in parallel rounds. Students can freely select the same course in BIDDING1 and BIDDING2. Only previously enrolled and conflicting courses are disabled.

### 4. `bidding-web/src/features/bidding/utils/__tests__/validation.test.ts`
- **Test Coverage**: Updated unit tests to reflect the new object-based signature and added specific test cases for enrollment and unavailability blocking.

## Behavioral Summary

| **Add/Drop Dropdown** | ❌ Disabled ("Previously Enrolled") | N/A | ❌ Disabled ("Already Enrolled" / "Already Waitlisted in this campaign") |
| **Bidding Dropdown** | ❌ Disabled ("Previously Enrolled") | ✅ Selectable | N/A |
| **`validateCourseAddition()`** | ❌ Blocked (safety net) | N/A | ❌ Blocked (safety net) |

## Impact
- **Proactive Feedback**: Students immediately see enrolled courses are disabled in the Add/Drop dropdown — no more late-stage hard block errors after save.
- **Correct Priority**: `getUnavailableReason()` now shows "Previously Enrolled" before any other condition (credit exceeded, conflict, etc.) for enrolled courses.
- **Bidding Phase Unaffected**: The bidding round dropdown does not block courses bid in parallel rounds — students can freely bid on the same course across BIDDING1 and BIDDING2.
- **Consistency**: Centralized business rules for course addition ensuring uniform behavior.
- **Stability**: Fixed type mismatches that were causing build failures.

## Testing / Verification Steps
1. Open the **Add/Drop & Waitlist** page.
2. Open the "Add Courses" dropdown.
3. Verify that courses already enrolled in show as **disabled** with "Previously Enrolled" text.
4. Confirm these courses **cannot be selected** (clicking does nothing).
5. Open the **Bidding** page for a parallel round (e.g., BIDDING2).
6. Verify that a course already bid on in BIDDING1 is **still selectable** in BIDDING2.
7. Run `npm run test` to verify all validation logic passes.
