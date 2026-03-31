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

## 3. Regression Test Coverage

- [x] 3.1 Unresolved Duplicate Constraints tests: Add tests in `AddDropValidatorPreviousEnrollmentTest` for parallel bidding duplicate blocking, successful duplicate resolution via drop, partial resolution failures, and clean passes without duplicates.
- [x] 3.2 Add/Drop Null-Safety tests: Add tests in `AddDropServiceNullSafetyTest.php` to verify summary fallback, audit log `oldData` fallback, and end-to-end safe progression with null `studentData`.
- [x] 3.3 Capital validation: Add/update `AddDropValidatorPreviousEnrollmentTest` scenarios for null `studentData` bid-point passes, negative capital rejection, and drop-refund offset successes.
- [x] 3.4 Cross-Module Duplicates: Add tests for `validateNoDuplicateCoursesWithCurrentEnrollment()` cross-module detection, same-submission duplicate rejection, and drop-then-add parity.

## 4. Parallel Bidding Submission Validation (New)

- [x] 4.1 Provide new query `findSubmittedCourseIdsInParallelRoundsByStudentAndProgram` in `bidding-api/src/Repository/BidRepository.php` to fetch course IDs that the given student has submitted across currently running parallel bidding rounds in the current Program (excluding the current module ID). Use `BidStatus::SELECTED` to filter submitted bids (not `BidStatus::SUBMITTED` which does not exist in the enum).
- [x] 4.2 Add `validateNoParallelRoundDuplicates(array $newBids, Student $student, Campaign $campaign, int $activeModuleId)` to `bidding-api/src/Domain/Campaign/ActiveCampaign/Validator/BidValidator.php`.
- [x] 4.3 Invoke `validateNoParallelRoundDuplicates()` from within `BidValidator->validate()` to reject the overall bid submission if cross-round duplicates are detected.
- [x] 4.4 Bidding cross-round duplicates: Add test scenarios to `BidValidatorTest` to cover parallel bidding duplicate submission prevention.

## 5. Verification

- [x] 5.1 Run PHPUnit tests covering `AddDropServiceNullSafetyTest.php` and `AddDropValidatorPreviousEnrollmentTest.php`.
- [x] 5.2 Manually verify parallel bidding duplicate forces drop.
- [x] 5.3 Manually verify Add/Drop 1 course blocked in Add/Drop 2.
- [x] 5.4 Manually verify negative capital submission is blocked.
- [x] 5.5 Manually verify dropping courses exclusively targets the submitted bidding round via QA video assessment.
- [x] 5.6 Manually verify submitting a course in BIDDING1 restricts the same course submission in BIDDING2.
- [x] 5.7 Fix `BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()`: replace undefined `BidStatus::SUBMITTED` with `BidStatus::SELECTED` to resolve 500 Internal Server Error on bid submission.
- [x] 5.8 Fix `BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()`: replace incorrect DQL field `b.moduleId` with `b.campaignModule` — `Bid` entity has no `moduleId` field/association, causing `[Semantical Error] line 0, col 200 near 'moduleId != '`.
