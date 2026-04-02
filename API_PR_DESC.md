# Fix: Duplicate Course Verification, Enrollment Validation & Capital Null Safety in Bidding and Add/Drop

## Problem
Jira: [DPBFAD-792](https://insead.atlassian.net/browse/DPBFAD-792)

Multiple validation and data integrity issues existed across the Bidding and Add/Drop & Waitlist phases:

### 1. Null-Dereference on Missing StudentData
The API directly dereferenced `Student::getStudentData()` without null checks. For students with missing data, this caused fatal runtime errors (`Call to a member function getRemainingCapital() on null`) blocking Add/Drop submission.

### 2. Unresolved Parallel Bidding Duplicates — Cross-Module Deadlock
After parallel bidding + simulation commit, students legitimately have the same course ENROLLED in both bidding1 and bidding2. `findDuplicateEnrolledCoursesByStudentAndCampaign()` accepted a `$moduleId` parameter but **never used it** in the query — it always queried ALL modules. This caused `validateNoUnresolvedDuplicateEnrollments()` to detect cross-module enrollments as "duplicates," throwing: *"You are enrolled in this course from multiple bidding rounds."* The student could only manage courses in the current module's add/drop UI and could not resolve the "duplicate" from another module — creating a permanent deadlock.

### 3. Capital (Bid Points) Check Disabled
`validateBidPoints()` existed in `AddDropValidator` but was never called in `AddDropService::submitAddDrop()`. This allowed submissions resulting in negative bid point balances.

### 4. Module-Scoped Drop Targeting Flaw
Dropping courses in one parallel bidding round erroneously targeted enrollments from another round. Drop queries and point refund calculations fetched the first available bid without scoping to the active module.

### 5. Enrolled Courses Not Disabled in Add/Drop Dropdown
Courses already enrolled were not blocked in the course selection dropdown. Students could select these courses and only encounter a hard error after submitting.

### 6. Undefined BidStatus::SUBMITTED in Cross-Round Query
`BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` referenced `BidStatus::SUBMITTED`, which does not exist in the `BidStatus` enum, causing a 500 Internal Server Error on bid submission.

### 7. Incorrect DQL Field `b.moduleId`
Same query referenced `b.moduleId` instead of `b.campaignModule`, causing a Doctrine Semantical Error.

### 8. Cross-Round Bidding Duplicate Prevention Was Incorrect
Students were blocked from bidding on the same course in parallel bidding rounds. This was wrong — students should be allowed to bid on the same course across multiple parallel rounds.

### 9. Waitlist Validation Gap
`AddDropValidator` ignored the `$waitlist` array in duplicate checks. `BidRepository::findEnrolledOrWaitlistedCourseIdsByStudentAndCampaign` filtered waitlists by current module, hiding cross-module waitlist duplicates.

### 10. Bidding Dropdown Incorrectly Marks Courses as "Previously Enrolled"
`getAvailableCourses()` computed `$allEnrolledCourseIds` without excluding the current campaign. Courses with `SELECTED` status from parallel bidding modules appeared as `is_enrolled = true`, disabling them as "Previously Enrolled" in other rounds' dropdowns.

## Changes

### 1. `AddDropService.php` — Null-Safe Financial Snapshot + Capital Enforcement
- Added `getStudentFinancialSnapshot(Student $student)` with fallback defaults (`credits_taken=0.0`, `remaining_capital=0`).
- Updated response summary and audit log payload to use snapshot fallback.
- Enabled `validateBidPoints()` call during submission.
- Updated `findOneBy` queries to filter by active module ID for drop operations.
- All validator calls now pass `$waitlist` array and `$moduleId` for proper scoping.

### 2. `AddDropValidator.php` — Duplicate Enrollment Validation
- Added `validateNoUnresolvedDuplicateEnrollments()` — detects within-module duplicate enrollments and requires resolution via drops. Cross-module enrollments are independent and not flagged.
- Updated `validateNoDuplicateCoursesWithCurrentEnrollment()` to accept `$moduleId` — drop exclusion is module-scoped so cross-module drops cannot bypass the duplicate check.
- Updated `validateNoDuplicateCoursesInSubmission`, `validateNoPreviousEnrollment`, and `validateNoDuplicateCoursesWithCurrentEnrollment` to include `$waitlist` array.
- `validateBidPoints()` uses null-safe remaining-capital access (`?? 0`).

### 3. `BidRepository.php` — Query Fixes
- **Critical Fix**: `findDuplicateEnrolledCoursesByStudentAndCampaign()` — added `WHERE b.campaign_module_id = :moduleId` clause when `$moduleId` is provided. This scopes duplicate detection to the current module, preventing cross-module deadlocks.
- Fixed `findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` — replaced `BidStatus::SUBMITTED` with `BidStatus::SELECTED` and `b.moduleId` with `b.campaignModule`.
- Broadened `findEnrolledOrWaitlistedCourseIdsByStudentAndCampaign` to return campaign-wide waitlists for cross-module duplicate detection.

### 4. `BidValidator.php` — Relaxed Bidding Phase Validation
- Removed `validateNoParallelRoundDuplicates()` call — students can now bid on the same course across parallel bidding rounds.
- Updated `validateNoDuplicates()` to allow same-course-different-section if one is a backup.
- Corrected config key from `min_bids_entire_round` to `min_capital_per_student` in `validateCapital()`.
- Integrated `AuditLogService` for late submission logging.

### 5. `StudentActiveCampaignController.php` — Fix `is_enrolled` Flag
- In `getAvailableCourses()`, passed `$campaign` as exclusion parameter to the `$allEnrolledCourseIds` query. This ensures courses from parallel bidding modules in the same campaign are not flagged as `is_enrolled`.

### 6. `AddDropAvailableCourseService.php` — UI-Level Duplicate Prevention
- Courses enrolled/waitlisted in other campaign modules are disabled in the dropdown with `already_enrolled_in_campaign` / `already_waitlisted_in_campaign` reason.

### 7. Test Coverage
- `AddDropValidatorPreviousEnrollmentTest` — parallel bidding duplicate blocking, resolution via drop, partial resolution failures, cross-phase duplicate tests, capital validation scenarios.
- `AddDropServiceNullSafetyTest` — summary fallback, audit log fallback, null `studentData` progression.

## Impact
- **Affected apps:** `bidding-api`
- **Affected entities:** `Student`, `StudentData`, `AuditLog`, `Bid`
- **Affected services:** `AddDropService`, `AddDropValidator`, `BidValidator`, `BidRepository`, `AddDropAvailableCourseService`
- **Affected controllers:** `StudentActiveCampaignController`
- **API contract:** No breaking changes. UI blocking uses existing enrollment state info.
- **Database migration:** None required.
- **Backward compatibility:** Preserved. Safely degrades to zero values instead of crashing. Previously-valid submissions with duplicates/negative capital are now correctly rejected.
- **Bidding round behavior:** Cross-round duplicate prevention REMOVED from bidding phase. Students can freely bid on the same course in multiple parallel rounds.

## Verification Steps
1. Create a new campaign and set the bidding phase open.
2. Do bidding from 3 different students across parallel bidding rounds (same courses in both rounds).
3. Close the bidding rounds.
4. Open simulation, commit simulation, close simulation.
5. Open the add/drop phase.
6. Impersonate a student who bid on the same course in both rounds.
7. Submit add/drop → should succeed without "multiple bidding rounds" error.
8. Verify students CAN bid on the same course in BIDDING1 and BIDDING2 without error.
9. Verify enrolled courses are disabled in the Add/Drop dropdown with "Previously Enrolled" label.
10. Verify negative capital submissions are blocked.
11. Verify dropping courses targets only the current bidding round module.
12. Verify courses from a prior campaign still show as "Previously Enrolled" in bidding dropdowns.
13. Run PHPUnit tests: `AddDropServiceNullSafetyTest` and `AddDropValidatorPreviousEnrollmentTest`.
