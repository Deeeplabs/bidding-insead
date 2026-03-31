# Duplicate Course Verification & Enrollment Capital Null Safety in Bidding and Add/Drop

## Summary
Jira: 
- https://insead.atlassian.net/browse/DPBFAD-792
- https://insead.atlassian.net/browse/DPBFAD-896

This PR addresses critical validation gaps and runtime crash risks across the Bidding and Add/Drop & Waitlist phases:
1. Prevents `Call to a member function getRemainingCapital() on null` errors when a `StudentData` record is missing.
2. Enforces mandatory resolution of pre-existing duplicate course enrollments stemming from parallel bidding rounds during Add/Drop.
3. Enables strict capital (bid points) validation which was previously bypassed, blocking negative capital submissions.
4. Aligns and documents existing duplicate-course validation guardrails (same-request duplicates) with test coverage.
5. Removes cross-round duplicate prevention from the Bidding phase — students are now allowed to submit bids for the same course across multiple active parallel bidding rounds (e.g., BIDDING1 and BIDDING2). Duplicate resolution is deferred to the Add/Drop phase.
6. Fixes a fatal error caused by referencing the undefined `BidStatus::SUBMITTED` enum case in the cross-round duplicate query, which caused a 500 Internal Server Error on every bid submission.
7. Fixes a Doctrine `[Semantical Error]` caused by referencing non-existent `b.moduleId` DQL field instead of the correct `b.campaignModule` association in the cross-round duplicate query.
8. Relaxes backup selection validation to allow students more flexibility when picking alternative courses.
9. Fixes the bidding dropdown incorrectly marking courses as "Previously Enrolled" across parallel bidding rounds — courses bid on in BIDDING1 were blocked in the BIDDING2 dropdown due to the `is_enrolled` flag including same-campaign bids.

## Problem Context

### 1. Null-Dereference on Missing StudentData
During processing (including summary calculations and audit logging), the API directly dereferenced `Student::getStudentData()` without null checks. For students with missing data, this caused fatal runtime errors blocking Add/Drop submission.

### 2. Unresolved Parallel Bidding Duplicates
In parallel bidding campaigns (multiple modules), students can legitimately end up with duplicate ENROLLED bids for the same course. However, there was no validation forcing them to drop one before performing Add/Drop manipulations.

### 3. Capital (Bid Points) Check Disabled
`AddDropValidator::validateBidPoints()` had correct logic but was never called in `AddDropService::submitAddDrop()`. This loophole allowed submissions resulting in negative bid point balances.

### 4. Cross-Round Bidding Duplicates (Revised — No Longer Blocked)
Previously, there was no cross-module evaluation of bids to prevent the same course from being submitted in parallel bidding rounds. This was initially planned as a validation block, but per business requirements, students **should** be allowed to bid on the same course in multiple parallel bidding rounds (e.g., BIDDING1 and BIDDING2). The only restrictions during the Bidding phase are:
- Cannot bid on courses the student is **already enrolled in** from prior campaigns.
- Cannot bid on **time-conflicting** courses.

### 5. Undefined BidStatus::SUBMITTED in Cross-Round Query
`BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` referenced `BidStatus::SUBMITTED`, which does not exist in the `BidStatus` enum. PHP throws a fatal `Error` for undefined enum cases before the null-coalescing (`??`) fallback can execute, resulting in a 500 Internal Server Error whenever a student attempts to submit bids during the Bidding phase.

### 6. Incorrect DQL Field Reference `b.moduleId` in Cross-Round Query
`BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` used `b.moduleId` in the DQL `andWhere` clause, but the `Bid` entity has no field or association named `moduleId`. The correct Doctrine association field is `campaignModule` (mapped to column `campaign_module_id`). This caused a `[Semantical Error] line 0, col 200 near 'moduleId != '` on every bid submission when a parallel module exclusion was applied.

### 7. Bidding Dropdown `is_enrolled` Flag Incorrectly Set for Parallel Rounds
In `StudentActiveCampaignController::getAvailableCourses()`, the `$allEnrolledCourseIds` query called `findEnrolledCourseIdsByStudentAndProgram($student, $program)` **without** excluding the current campaign. This caused the `is_enrolled` flag to be set to `true` for courses with `SELECTED` bids in the same campaign's other bidding modules. The frontend's `getUnavailableReason()` treated `is_enrolled = true` as "Previously Enrolled", disabling those courses in the bidding round 2 dropdown — even though students should be free to bid on the same courses across parallel rounds.

## What Changed

1. **Null-safe financial snapshot in `AddDropService`**
   - Added `getStudentFinancialSnapshot(Student $student)` which provides safe defaults (`credits_taken = 0.0`, `remaining_capital = 0`).
   - Applied in `buildResponse()` for `bid_points_remaining` calculation and `createAuditLog()` for `oldData`.
   - Updated `AddDropValidator::validateBidPoints()` to read remaining capital safely via null-safe access: `$student->getStudentData()?->getRemainingCapital() ?? 0`.

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
   - Rejects adding multiple sections of the same course within a single submission (same-request duplicate detection).
   - Campaign-scoped cross-phase duplicate prevention via `validateNoDuplicateCoursesWithCurrentEnrollment($moduleId)`: rejects adding a course already enrolled/waitlisted in the campaign across any module. Drop-exclusion is now module-scoped — a class in the `drops` list only exempts its course if the student has an ENROLLED or SELECTED bid for that class **in the current module** (`campaignModule = $moduleId`). This closes the bypass where a cross-module class in drops could incorrectly exclude a course from the duplicate check.
   - **Waitlist Validation Gap Closed**: Updated `AddDropValidator` to include the `$waitlist` array in all duplicate and previous enrollment checks. Modified `BidRepository::findEnrolledOrWaitlistedCourseIdsByStudentAndCampaign` to return campaign-wide waitlists (removing module-level filtering for waitlist status). This ensures that students cannot bypass duplicate course rules by waitlisting for a course they are already involved with in another phase.
   - Added UI-level blocking via the `AddDropAvailableCourseDto` using `disabled_reason = 'already_enrolled_in_campaign'` and `'already_waitlisted_in_campaign'` for courses the student is already enrolled/waitlisted in within the current campaign (cross-module duplicate detection at the API response level).

6. **Cross-Round Duplicate Prevention REMOVED from Bidding Phase (`BidValidator`)**
   - Commented out the `validateNoParallelRoundDuplicates()` call in `BidValidator::validate()`.
   - Students are now **allowed** to submit bids for the same course across multiple parallel bidding rounds (e.g., BIDDING1 and BIDDING2). This is intentional — duplicate resolution occurs during the Add/Drop phase via `AddDropValidator::validateNoUnresolvedDuplicateEnrollments()`.
   - The `BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` query and `BidValidator::validateNoParallelRoundDuplicates()` method are retained in code but not called during bidding validation.

7. **Fix undefined `BidStatus::SUBMITTED` enum reference**
   - Replaced `BidStatus::SUBMITTED` with `BidStatus::SELECTED` in `BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()`. The `BidStatus` enum has no `SUBMITTED` case; bids during the Bidding phase are initially stored with status `SELECTED` (value `1`). The previous code caused a fatal PHP `Error` on every bid submission attempt.

8. **Fix incorrect DQL field `b.moduleId` → `b.campaignModule`**
   - Replaced `b.moduleId` with `b.campaignModule` in the exclude-module clause of `BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()`. The `Bid` entity has no `moduleId` property; the correct ManyToOne association is `campaignModule` (column `campaign_module_id`). This resolved the `[Semantical Error] line 0, col 200` that occurred on bid submission.

9. **Relaxed Backup Validation (`BidValidator`)**
   - Updated `validateNoDuplicates()` to allow multiple sections of the same course as long as one of them is a backup.
   - `validateTimeConflicts()` continues to skip backup bids when evaluating schedule overlaps.
   - `validateNoPreviousEnrollment()` remains enforced — students still cannot bid on previously enrolled courses from prior campaigns.

10. **Fix bidding dropdown `is_enrolled` flag for parallel rounds (`StudentActiveCampaignController`)**
    - In `getAvailableCourses()`, the `$allEnrolledCourseIds` query now passes `$campaign` as the exclusion parameter (matching `$previouslyEnrolledCourseIds`).
    - Previously, `$allEnrolledCourseIds` included courses from ALL campaigns (no exclusion), causing courses with `SELECTED` bids in the same campaign's other modules (e.g., BIDDING1) to have `is_enrolled = true` when viewing BIDDING2.
    - The frontend's `getUnavailableReason()` in `validation.util.ts` treats `is_enrolled = true` as "Previously Enrolled", disabling those courses in the dropdown.
    - With this fix, only courses from **prior campaigns** are flagged as `is_enrolled`, while courses from parallel bidding rounds in the same campaign remain freely selectable.

11. **Refactor and Fix `BidValidator` Capital Logic**
    - Corrected the configuration key from `min_bids_entire_round` to `min_capital_per_student` in `validateCapital()`. The previous key was mismatched with the actual campaign configuration.
    - Simplified `validateCapital()` signature by removing unused `$moduleId` and `$student` parameters.
    - Improved type safety by casting `bidPoints` to integer during summation.

## Bidding Phase Validation Summary

| Validation | Primary Bids | Backup Bids |
|---|---|---|
| Previously enrolled courses | ❌ Blocked | ❌ Blocked |
| Time conflicts | ❌ Blocked | ✅ Allowed |
| Same course across parallel rounds | ✅ Allowed | ✅ Allowed |
| Same course in single submission | ❌ Blocked | ✅ Allowed (different section) |

## Impact & Behavioral Changes

- **API response & Database Schema:** Unchanged.
- **Null Safety:** Missing `studentData` degrades gracefully to `0` values rather than throwing HTTP 500s.
- **Capital Constraints:** Users can no longer exploit Add/Drop with underfunded capital.
- **Duplicates (Add/Drop):** Students entering Add/Drop with multi-round (parallel bidding) duplicates are forced to resolve them (scoped per module) before any new changes apply. Campaign-scoped cross-phase duplicate prevention (same course in Add/Drop 1 and Add/Drop 2) is fully enforced — the cross-module drop bypass has been closed by module-scoping the drop-exclusion logic in `validateNoDuplicateCoursesWithCurrentEnrollment()`.
- **Duplicates (Bidding):** Students **CAN** now bid on the same course in multiple parallel bidding rounds. Cross-round duplicate prevention is removed from the bidding phase. Duplicate resolution is deferred to the Add/Drop phase.

## Tests Added/Updated

1. `AddDropServiceNullSafetyTest.php`: Summary and audit fallback validation with null `studentData`.
2. `AddDropValidatorPreviousEnrollmentTest.php`:
   - Parallel bidding duplicate blockage + resolution by dropping.
   - Null-`studentData` bid-points pass/fail boundaries.
   - Insufficient capital rejections vs. pass-via-drop-refund.
   - Same-submission duplicate course rejection.
   - Cross-phase module-scoped tests for `validateNoDuplicateCoursesWithCurrentEnrollment($moduleId)`: cross-module duplicate rejection, cross-module drop bypass rejection, same-module drop-then-add acceptance, and different-course pass.

## Verification

- [x] All entry point controllers verified conceptually.
- [x] PHPUnit suite tests run and passed cleanly.
- [x] Fix verified: `BidStatus::SUBMITTED` reference removed, replaced with `BidStatus::SELECTED` — bid submissions no longer produce 500 errors.
- [x] Fix verified: `b.moduleId` replaced with `b.campaignModule` — Doctrine semantical error on bid submission resolved.
- [x] Verified backup flexibility: students can now add the same course in different sections if one is marked as backup.
- [x] Verified cross-round duplicate prevention is removed: students CAN submit the same course in BIDDING1 and BIDDING2 without error.
- [x] Verified previously enrolled course blocking remains in the bidding phase.
- [ ] Verified bidding dropdown fix: courses bid on in BIDDING1 (e.g., 'Art of Why', 'Blue Ocean Strategy') are NOT shown as "Previously Enrolled" in the BIDDING2 dropdown.
- [ ] Verified courses genuinely enrolled from a prior campaign still show as "Previously Enrolled" in both BIDDING1 and BIDDING2 dropdowns.
- [x] Regression fixed: Add/Drop 1 course now correctly blocked in Add/Drop 2 even when a cross-module class is included in the `drops` list — `validateNoDuplicateCoursesWithCurrentEnrollment()` now requires a matching enrolled bid in the current module before honoring a drop exclusion.
