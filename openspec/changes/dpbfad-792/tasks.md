## 1. Root Cause & Null-Safety Implementation

- [x] 1.1 Add `getStudentFinancialSnapshot(Student $student)` in `bidding-api/src/Domain/Campaign/ActiveCampaign/AddDropService.php` with fallback defaults (`credits_taken=0.0`, `remaining_capital=0`).
- [x] 1.2 Update Add/Drop response summary in `AddDropService::buildResponse()` to use snapshot fallback.
- [x] 1.3 Update `AddDropService::createAuditLog()` old-data payload to use snapshot fallback.
- [x] 1.4 Update `AddDropValidator::validateBidPoints()` to use null-safe remaining-capital access (`?? 0`).

## 2. Duplicate Enrollment Validation & Capital Enforcement

- [x] 2.1 Add `findDuplicateEnrolledCoursesByStudentAndCampaign()` to `bidding-api/src/Repository/BidRepository.php` to group and return `course_id`s with >1 ENROLLED/SELECTED bids.
- [x] 2.2 Add `validateNoUnresolvedDuplicateEnrollments(array $drops, Student $student, Campaign $campaign, ?int $moduleId = null)` to `AddDropValidator.php`. Ensure it calculates remaining enrollment count after factoring in drops.
- [x] 2.3 Call `validateNoUnresolvedDuplicateEnrollments()` in `AddDropService::submitAddDrop()` immediately after student eligibility check (Step 4a). Pass `$moduleId` to scope validation to current bidding round.
- [x] 2.4 Enable `validateBidPoints()` in `AddDropService::submitAddDrop()` (Step 8a) and remove the comment stating it is not enforced.
- [x] 2.5 Update `AddDropService.php` and `AddDropValidator.php` to pass `$moduleId` to limit finding dropped bids and refunded bid points exclusively to that specific module.
- [x] 2.6 **Critical Fix**: Ensure `validateNoUnresolvedDuplicateEnrollments()` receives `$moduleId` so that Add/Drop in bid2 does NOT see duplicates from bid1. The validation is now properly scoped per module.
- [x] 2.7 **Cross-Phase Duplicate Fix**: Update `validateNoDuplicateCoursesWithCurrentEnrollment()` signature to accept `?int $moduleId = null`. When `$moduleId` is provided, build `$droppedCourseIds` by querying the database for each dropped class — only include the course in the exclusion map if the student has an ENROLLED or SELECTED bid for that class in the **current module** (`campaignModule = $moduleId`). If no matching bid exists in the current module, the course is NOT excluded from the campaign-wide duplicate check.
- [x] 2.8 **Pass moduleId at call site**: In `AddDropService::submitAddDrop()` Step 5c, pass `$moduleId` to `validateNoDuplicateCoursesWithCurrentEnrollment($enrollments, $drops, $student, $campaign, $moduleId)` so the module-scoped drop exclusion logic activates.

## 3. Regression Test Coverage

- [x] 3.1 Unresolved Duplicate Constraints tests: Add tests in `AddDropValidatorPreviousEnrollmentTest` for parallel bidding duplicate blocking, successful duplicate resolution via drop, partial resolution failures, and clean passes without duplicates.
- [x] 3.2 Add/Drop Null-Safety tests: Add tests in `AddDropServiceNullSafetyTest.php` to verify summary fallback, audit log `oldData` fallback, and end-to-end safe progression with null `studentData`.
- [x] 3.3 Capital validation: Add/update `AddDropValidatorPreviousEnrollmentTest` scenarios for null `studentData` bid-point passes, negative capital rejection, and drop-refund offset successes.
- [x] 3.4 Cross-Phase Add/Drop Duplicate Tests: Add tests for `validateNoDuplicateCoursesWithCurrentEnrollment()` covering:
  - Student enrolls in Finance 101 in Add/Drop 1 (moduleId=3), then tries to enroll Finance 101 class in Add/Drop 2 (moduleId=5) → REJECTED
  - Student enrolls in Finance 101 in Add/Drop 1 (moduleId=3), then tries to enroll a DIFFERENT SECTION of Finance 101 in Add/Drop 2 (moduleId=5) → REJECTED
  - Student enrolls Finance 101 in Add/Drop 1 (moduleId=3), then submits Add/Drop 2 (moduleId=5) with Finance 101 class in drops list (cross-module bypass attempt) + Finance 101 EB in enrollments → REJECTED (cross-module drop does not count as exclusion)
  - Legitimate drop-then-add within SAME module (moduleId=3 drops class 1501 and adds class 1502 of same course) → ACCEPTED
  - Same-submission duplicate rejection (two sections of same course in one request) → REJECTED

## 4. Bidding Round Validation Changes

- [x] 4.1 **Remove cross-round duplicate prevention from bidding**: Remove the `validateNoParallelRoundDuplicates()` call from `BidValidator::validate()`. Students are allowed to submit the same bids for the same courses in 2 or more active parallel bidding rounds.
- [x] 4.2 **Relax Backup Validation**: Update `BidValidator::validateNoDuplicates()` to allow same-course-different-section if one is a backup.
- [x] 4.3 **Relax Backup Validation**: Ensure `validateTimeConflicts()` continues to skip backup bids when evaluating schedule overlaps.
- [x] 4.4 **Config Fix**: Correct `min_capital_per_student` key mismatch in `BidValidator.php`.
- [x] 4.5 **Keep Previously Enrolled Blocking**: Confirm `validateNoPreviousEnrollment()` remains in `BidValidator::validate()` — students cannot bid on courses they have already completed/enrolled in from prior campaigns.
- [x] 4.6 Fix `BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()`: replace undefined `BidStatus::SUBMITTED` with `BidStatus::SELECTED`.
- [x] 4.7 Fix `BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()`: replace incorrect DQL field `b.moduleId` with `b.campaignModule`.

## 5. UI — Add/Drop Dropdown Enrolled Course Disabling

- [x] 5.1 **Dropdown Disabling**: In `use-course-options.ts`, ensure courses with `is_enrolled === true` are disabled with "Previously Enrolled" reason in the Add/Drop phase dropdown. This prevents selection at the UI level.
- [x] 5.2 **`getUnavailableReason` priority**: Ensure the `is_enrolled` check in `getUnavailableReason()` fires before other conditions (credit, full, fallback) so enrolled courses are always disabled regardless of seat availability.
- [x] 5.3 **Safety net validation**: In `validation.util.ts`, `validateCourseAddition()` checks `is_enrolled` and `unavailable_reason` on the `AvailableCourse` object. If a course bypasses dropdown disabling, this validation blocks it with a toast error.
- [x] 5.4 **Toast feedback**: In `use-add-drop-waitlist-form.tsx`, `handleAddCourse` calls `validateCourseAddition()` and shows a toast error for invalid courses (including enrolled).
- [x] 5.5 **Bidding round dropdown**: Confirm the bidding form dropdown (`use-bidding-form.ts`) does NOT disable courses that are bid on in parallel rounds. Only previously enrolled and conflicting courses are disabled.

## 6. Cross-Phase Add/Drop Duplicate — Targeted Verification

- [x] 6.12 Manually verify: student submits Finance 101 in Add/Drop 1 (moduleId=bid1) and then submits Finance 101 again in Add/Drop 2 (moduleId=bid2) → API returns 400 with "Already enrolled in course Finance 101 in this campaign."
- [x] 6.13 Manually verify: student submits Finance 101 Section EA in Add/Drop 1, then submits Finance 101 Section EB in Add/Drop 2 → API returns 400 (different section, same course, must still be rejected).
- [x] 6.14 Manually verify the cross-module drop bypass is blocked: student submits Add/Drop 2 with Finance 101 class from Add/Drop 1 in the drops list and Finance 101 Section EB in enrollments → still 400 (cross-module drop is not counted as a valid exclusion).
- [x] 6.15 Manually verify: drop-then-add Finance 101 EA → EB within the SAME module still works after the fix (moduleId in drops and enrollments match).
- [x] 6.16 Manually verify waitlist duplicate: student waitlisted for Course A in Module 1, then attempts to waitlist for Course A in Module 2 → REJECTED with "Already waitlisted for this course in this campaign."

## 7. Waitlist Validation Implementation

- [x] 7.1 Update `AddDropValidator` to include `$waitlist` array in `validateNoDuplicateCoursesInSubmission`, `validateNoPreviousEnrollment`, and `validateNoDuplicateCoursesWithCurrentEnrollment`.
- [x] 7.2 Update `AddDropService::submitAddDrop` to pass `$waitlist` to validators and trigger validation for waitlist-only requests.
- [x] 7.3 Broaden `BidRepository::findEnrolledOrWaitlistedCourseIdsByStudentAndCampaign` to return campaign-wide waitlists for cross-module duplicate detection.
- [x] 7.4 Verify UI correctly disables courses waitlisted in other modules using the updated repository query.

## 8. Original Verification

- [x] 8.1 Run PHPUnit tests covering `AddDropServiceNullSafetyTest.php` and `AddDropValidatorPreviousEnrollmentTest.php`.
- [x] 8.2 Manually verify parallel bidding duplicate forces drop in Add/Drop.
- [x] 8.3 Manually verify Add/Drop 1 course blocked in Add/Drop 2. (**REGRESSION** — fixed via tasks 2.7 & 2.8; `validateNoDuplicateCoursesWithCurrentEnrollment` now module-scoped; cross-module drop bypass closed)
- [x] 8.4 Manually verify negative capital submission is blocked.
- [x] 8.5 Manually verify dropping courses exclusively targets the submitted bidding round.
- [x] 8.6 Manually verify students CAN submit the same course in BIDDING1 and BIDDING2 without error.
- [x] 8.7 Verify students can add the same course with different sections in backup.
- [x] 8.8 Verify no validation error for time conflicts with primary courses for backups.
- [x] 8.9 Verify UI correctly disables previously enrolled courses in the Add/Drop dropdown (cannot be selected).
- [x] 8.10 Verify UI shows "Previously Enrolled" label on disabled courses in Add/Drop dropdown.
- [x] 8.11 Verify build errors in `bidding-web` are resolved and type safety is enforced.
