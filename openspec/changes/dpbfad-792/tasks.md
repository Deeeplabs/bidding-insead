## 1. Root Cause & Null-Safety Implementation

- [x] 1.1 Add `getStudentFinancialSnapshot(Student $student)` in `bidding-api/src/Domain/Campaign/ActiveCampaign/AddDropService.php` with fallback defaults (`credits_taken=0.0`, `remaining_capital=0`).
- [x] 1.2 Update Add/Drop response summary in `AddDropService::buildResponse()` to use snapshot fallback.
- [x] 1.3 Update `AddDropService::createAuditLog()` old-data payload to use snapshot fallback.
- [x] 1.4 Update `AddDropValidator::validateBidPoints()` to use null-safe remaining-capital access (`?? 0`).

## 2. Duplicate Enrollment Validation & Capital Enforcement

- [x] 2.1 Add `findDuplicateEnrolledCoursesByStudentAndCampaign()` to `bidding-api/src/Repository/BidRepository.php` to group and return `course_id`s with >1 ENROLLED/SELECTED bids.
- [x] 2.2 Add `validateNoUnresolvedDuplicateEnrollments(array $drops, Student $student, Campaign $campaign)` to `AddDropValidator.php`. Ensure it calculates remaining enrollment count after factoring in drops.
- [x] 2.3 Call `validateNoUnresolvedDuplicateEnrollments()` in `AddDropService::submitAddDrop()` immediately after student eligibility check (Step 4a).
- [x] 2.4 Enable `validateBidPoints()` in `AddDropService::submitAddDrop()` (Step 8a) and remove the comment stating it is not enforced.

## 3. Regression Test Coverage

- [x] 3.1 Unresolved Duplicate Constraints tests: Add tests in `AddDropValidatorPreviousEnrollmentTest` for parallel bidding duplicate blocking, successful duplicate resolution via drop, partial resolution failures, and clean passes without duplicates.
- [x] 3.2 Add/Drop Null-Safety tests: Add tests in `AddDropServiceNullSafetyTest.php` to verify summary fallback, audit log `oldData` fallback, and end-to-end safe progression with null `studentData`.
- [x] 3.3 Capital validation: Add/update `AddDropValidatorPreviousEnrollmentTest` scenarios for null `studentData` bid-point passes, negative capital rejection, and drop-refund offset successes.
- [x] 3.4 Cross-Module Duplicates: Add tests for `validateNoDuplicateCoursesWithCurrentEnrollment()` cross-module detection, same-submission duplicate rejection, and drop-then-add parity.

## 4. Verification

- [x] 4.1 Run PHPUnit tests covering `AddDropServiceNullSafetyTest.php` and `AddDropValidatorPreviousEnrollmentTest.php`.
- [x] 4.2 Manually verify parallel bidding duplicate forces drop.
- [x] 4.3 Manually verify Add/Drop 1 course blocked in Add/Drop 2.
- [x] 4.4 Manually verify negative capital submission is blocked.
