# Duplicate Course Verification & Enrollment Capital Null Safety in Add/Drop

## Summary
Jira: https://insead.atlassian.net/browse/DPBFAD-792

This PR addresses critical validation gaps and runtime crash risks in the Add/Drop & Waitlist phase:
1. Prevents `Call to a member function getRemainingCapital() on null` errors when a `StudentData` record is missing.
2. Enforces mandatory resolution of pre-existing duplicate course enrollments stemming from parallel bidding rounds.
3. Enables strict capital (bid points) validation which was previously bypassed, blocking negative capital submissions.
4. Aligns and documents existing duplicate-course validation guardrails (cross-module and same-request duplicates) with robust test coverage.

## Problem Context

### 1. Null-Dereference on Missing StudentData
During processing (including summary calculations and audit logging), the API directly dereferenced `Student::getStudentData()` without null checks. For students with missing data, this caused fatal runtime errors blocking Add/Drop submission.

### 2. Unresolved Parallel Bidding Duplicates
In parallel bidding campaigns (multiple modules), students can legitimately end up with duplicate ENROLLED bids for the same course. However, there was no validation forcing them to drop one before performing Add/Drop manipulations.

### 3. Capital (Bid Points) Check Disabled
`AddDropValidator::validateBidPoints()` had correct logic but was never called in `AddDropService::submitAddDrop()`. This loophole allowed submissions resulting in negative bid point balances.

## What Changed

1. **Null-safe financial snapshot in `AddDropService`**
   - Added `getStudentFinancialSnapshot(Student $student)` which provides safe defaults (`credits_taken = 0.0`, `remaining_capital = 0`).
   - Applied in `buildResponse()` for `bid_points_remaining` calculation and `createAuditLog()` for `oldData`.
   - Updated `AddDropValidator::validateBidPoints()` to read remaining capital safely via `$student->getStudentData()?->getRemainingCapital() ?? 0`.

2. **Unresolved Duplicate Prevention in `AddDropValidator` & `BidRepository`**
   - Added `BidRepository::findDuplicateEnrolledCoursesByStudentAndCampaign()` to query same-campaign duplicate entries.
   - Added `AddDropValidator::validateNoUnresolvedDuplicateEnrollments()` which mandates dropping all-but-one class per duplicated course.
   - Wired this validation symmetrically as step 4a in `submitAddDrop()` (runs unconditionally).

3. **Module-Scoped Add/Drop Resolution in `AddDropService`**
   - Added `$moduleId` scoping to `findOneBy()` queries during drop processing, point refunds (`validateBidPoints()`), and response building (`buildResponse()`).
   - Resolves a critical bug where dropping courses in one parallel bidding round (e.g. `bid2`) erroneously dropped identical course enrollments belonging to another round (e.g. `bid1`).

4. **Capital Check Enablement**
   - Uncommented/enabled `validateBidPoints()` inside `submitAddDrop()`.

5. **Duplicate-course Guardrails (Documented & Tested)**
   - Rejects adding multiple sections of the same course within singular submissions.
   - Checks campaign-scoped prior module additions (e.g., Add/Drop 1 courses cannot be added in Add/Drop 2 unless dropped).

## Impact & Behavioral Changes

- **API response & Database Schema:** Unchanged.
- **Null Safety:** Missing `studentData` degrades gracefully to `0` values rather than throwing HTTP 500s.
- **Capital Constraints:** Users can no longer exploit Add/Drop with underfunded capital.
- **Duplicates:** Students entering Add/Drop with multi-round duplicates are forced to resolve them before any new changes apply.

## Tests Added/Updated

1. `AddDropServiceNullSafetyTest.php`: Summary and audit fallback validation with null `studentData`.
2. `AddDropValidatorPreviousEnrollmentTest.php`:
   - Parallel bidding duplicate blockage + resolution by dropping.
   - Null-`studentData` bid-points pass/fail boundaries.
   - Insufficient capital rejections vs. pass-via-drop-refund.
   - Campaign-wide cross-module duplicate restrictions and same-submission duplication.

## Verification

- [x] All 4 entry point controllers verified conceptually (`StudentActiveCampaignController`, `CreateBiddingAddDropController`, `CampusExchangeAddDropWaitlistController`, `CampusExchangeAddDropAllotmentController`).
- [x] `cmd /c vendor\bin\phpunit tests\Unit\Domain\Campaign\ActiveCampaign\AddDropServiceNullSafetyTest.php tests\Unit\Domain\Campaign\ActiveCampaign\Validator\AddDropValidatorPreviousEnrollmentTest.php`
