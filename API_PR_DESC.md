# Duplicate Course Verification & Enrollment Capital Null Safety in Bidding and Add/Drop

## Summary
Jira: https://insead.atlassian.net/browse/DPBFAD-792

This PR addresses critical validation gaps and runtime crash risks across the Bidding and Add/Drop & Waitlist phases:
1. Prevents `Call to a member function getRemainingCapital() on null` errors when a `StudentData` record is missing.
2. Enforces mandatory resolution of pre-existing duplicate course enrollments stemming from parallel bidding rounds during Add/Drop.
3. Enables strict capital (bid points) validation which was previously bypassed, blocking negative capital submissions.
4. Aligns and documents existing duplicate-course validation guardrails (cross-module and same-request duplicates) with robust test coverage.
5. Introduces strict validation during the active Bidding phase to prevent students from submitting bids for the same exact course across multiple active parallel bidding rounds (e.g., BIDDING1 and BIDDING2).
6. Fixes a fatal error caused by referencing the undefined `BidStatus::SUBMITTED` enum case in the cross-round duplicate query, which caused a 500 Internal Server Error on every bid submission.
7. Fixes a Doctrine `[Semantical Error]` caused by referencing non-existent `b.moduleId` DQL field instead of the correct `b.campaignModule` association in the cross-round duplicate query.

## Problem Context

### 1. Null-Dereference on Missing StudentData
During processing (including summary calculations and audit logging), the API directly dereferenced `Student::getStudentData()` without null checks. For students with missing data, this caused fatal runtime errors blocking Add/Drop submission.

### 2. Unresolved Parallel Bidding Duplicates
In parallel bidding campaigns (multiple modules), students can legitimately end up with duplicate ENROLLED bids for the same course. However, there was no validation forcing them to drop one before performing Add/Drop manipulations.

### 3. Capital (Bid Points) Check Disabled
`AddDropValidator::validateBidPoints()` had correct logic but was never called in `AddDropService::submitAddDrop()`. This loophole allowed submissions resulting in negative bid point balances.

### 4. Direct Duplicate Submission in Parallel Bidding Modules
In a campaign with parallel bidding modules (BIDDING1, BIDDING2), users could independently submit bids for the identical course in both modules. There was no cross-module evaluation of open bids to halt duplicate course selections until after bids were computed to enrollments.

### 5. Undefined BidStatus::SUBMITTED in Cross-Round Query
`BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` referenced `BidStatus::SUBMITTED`, which does not exist in the `BidStatus` enum. PHP throws a fatal `Error` for undefined enum cases before the null-coalescing (`??`) fallback can execute, resulting in a 500 Internal Server Error whenever a student attempts to submit bids during the Bidding phase.

### 6. Incorrect DQL Field Reference `b.moduleId` in Cross-Round Query
`BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` used `b.moduleId` in the DQL `andWhere` clause, but the `Bid` entity has no field or association named `moduleId`. The correct Doctrine association field is `campaignModule` (mapped to column `campaign_module_id`). This caused a `[Semantical Error] line 0, col 200 near 'moduleId != '` on every bid submission when a parallel module exclusion was applied.

## What Changed

1. **Null-safe financial snapshot in `AddDropService`**
   - Added `getStudentFinancialSnapshot(Student $student)` which provides safe defaults (`credits_taken = 0.0`, `remaining_capital = 0`).
   - Applied in `buildResponse()` for `bid_points_remaining` calculation and `createAuditLog()` for `oldData`.
   - Updated `AddDropValidator::validateBidPoints()` to read remaining capital safely via `$student->getStudentData()?->getRemainingCapital() ?? 0`.

2. **Unresolved Duplicate Prevention in `AddDropValidator` & `BidRepository`**
   - Added `BidRepository::findDuplicateEnrolledCoursesByStudentAndCampaign()` to query same-campaign duplicate entries. Added optional `$moduleId` parameter to scope duplicates to a specific bidding round.
   - Added `AddDropValidator::validateNoUnresolvedDuplicateEnrollments()` which mandates dropping all-but-one class per duplicated course. Now accepts `$moduleId` to validate only within the current module (bid1 or bid2).
   - Wired this validation as step 4a in `submitAddDrop()` (runs unconditionally).
   - **Critical Fix**: The validation now properly scopes to the current bidding round using `$moduleId`, so Add/Drop in bid2 does NOT incorrectly block on duplicates from bid1.

3. **Module-Scoped Add/Drop Resolution in `AddDropService`**
   - Added `$moduleId` scoping to `findOneBy()` queries during drop processing, point refunds (`validateBidPoints()`), and response building (`buildResponse()`).
   - Resolves a critical bug where dropping courses in one parallel bidding round (e.g. `bid2`) erroneously dropped identical course enrollments belonging to another round (e.g. `bid1`).

4. **Capital Check Enablement**
   - Uncommented/enabled `validateBidPoints()` inside `submitAddDrop()`.

5. **Duplicate-course Guardrails (Documented & Tested)**
   - Rejects adding multiple sections of the same course within singular submissions.
   - Checks campaign-scoped prior module additions (e.g., Add/Drop 1 courses cannot be added in Add/Drop 2 unless dropped).

6. **Cross-Round Duplicate Submission Prevention (`BidValidator`)**
   - Implemented `BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` to query for courses actively bidded on in parallel bidding round modules.
   - Added `BidValidator::validateNoParallelRoundDuplicates()` evaluating prior PENDING selections to robustly block users from placing new requests on duplicated courses during the immediate Bidding Phase.

7. **Fix undefined `BidStatus::SUBMITTED` enum reference**
   - Replaced `BidStatus::SUBMITTED` with `BidStatus::PENDING` in `BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()`. The `BidStatus` enum has no `SUBMITTED` case; bids during the Bidding phase are stored with status `PENDING` (value `10`). The previous code caused a fatal PHP `Error` on every bid submission attempt.

8. **Fix incorrect DQL field `b.moduleId` → `b.campaignModule`**
   - Replaced `b.moduleId` with `b.campaignModule` in the exclude-module clause of `BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()`. The `Bid` entity has no `moduleId` property; the correct ManyToOne association is `campaignModule` (column `campaign_module_id`). This resolved the `[Semantical Error] line 0, col 200` that occurred on bid submission.

## Impact & Behavioral Changes

- **API response & Database Schema:** Unchanged.
- **Null Safety:** Missing `studentData` degrades gracefully to `0` values rather than throwing HTTP 500s.
- **Capital Constraints:** Users can no longer exploit Add/Drop with underfunded capital.
- **Duplicates:** Students entering Add/Drop with multi-round duplicates are forced to resolve them before any new changes apply. Furthermore, Bidding users are halted from generating duplications spanning multiple modules prior to simulation.

## Tests Added/Updated

1. `AddDropServiceNullSafetyTest.php`: Summary and audit fallback validation with null `studentData`.
2. `AddDropValidatorPreviousEnrollmentTest.php`:
   - Parallel bidding duplicate blockage + resolution by dropping.
   - Null-`studentData` bid-points pass/fail boundaries.
   - Insufficient capital rejections vs. pass-via-drop-refund.
   - Campaign-wide cross-module duplicate restrictions and same-submission duplication.
3. `BidValidatorPreviousEnrollmentTest.php`:
   - Enforce rejection upon detecting parallel bid submission on duplicated course across multiple modules.

## Verification

- [x] All entry point controllers verified conceptually.
- [x] PHPUnit suite tests run and passed cleanly.
- [x] Fix verified: `BidStatus::SUBMITTED` reference removed, replaced with `BidStatus::PENDING` — bid submissions no longer produce 500 errors.
- [x] Fix verified: `b.moduleId` replaced with `b.campaignModule` — Doctrine semantical error on bid submission resolved.
