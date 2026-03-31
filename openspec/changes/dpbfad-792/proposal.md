## Why

Seven issues exist across the Bidding and Add/Drop & Waitlist phases:

### 1. Null-Dereference on Missing StudentData
During processing (including summary calculations and audit logging), the API directly dereferenced `Student::getStudentData()` without null checks. For students with missing data, this caused fatal runtime errors (`Call to a member function getRemainingCapital() on null`) blocking Add/Drop submission.

### 2. Unresolved Parallel Bidding Duplicates
In parallel bidding campaigns (multiple modules), students can legitimately end up with duplicate ENROLLED bids for the same course. However, there was no validation forcing them to drop one before performing Add/Drop manipulations. `validateNoDuplicateCoursesWithCurrentEnrollment()` correctly blocks adding new duplicates, but doesn't force resolution of pre-existing ones.

### 3. Capital (Bid Points) Check Disabled
`validateBidPoints()` exists in `AddDropValidator` but was never called in `AddDropService::submitAddDrop()`. This loophole allowed submissions resulting in negative bid point balances.

### 4. Module-Scoped Drop Targeting Flaw
Dropping courses in one parallel bidding round (e.g., `bid1`) erroneously targeted and dropped identical course enrollments from another round (e.g., `bid1`). Drop queries and point refund calculations implicitly fetched the first available bid without scoping it to the active module.

### 5. Enrolled Courses Not Disabled in Add/Drop Dropdown
In the Add/Drop & Waitlist phase, courses that students are already enrolled in are not blocked or disabled in the course selection dropdown. Students can select these courses again and only encounter a hard block error after submitting/saving. Enrolled courses should be preemptively disabled in the dropdown at the UI level, preventing selection entirely.

### 6. Undefined BidStatus::SUBMITTED in Cross-Round Query
`BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` referenced `BidStatus::SUBMITTED`, which does not exist in the `BidStatus` enum. PHP throws a fatal `Error` for undefined enum cases before the null-coalescing (`??`) fallback can execute, resulting in a 500 Internal Server Error whenever a student attempts to submit bids during the Bidding phase.

### 8. Waitlist Validation Gap
- `AddDropValidator` completely ignored the `$waitlist` array when checking for duplicate course enrollments (current campaign) or previous enrollments (programme history).
- `BidRepository::findEnrolledOrWaitlistedCourseIdsByStudentAndCampaign` incorrectly filtered waitlisted courses by the current module, hiding cross-module waitlist duplicates.
- `AddDropService` skipped duplicate/capital validation entirely if only waitlist items were submitted.

## What Changes

1. **Force resolution of parallel bidding duplicates in Add/Drop**: Before processing any add/drop submission, detect if the student has multiple ENROLLED/SELECTED bids for the same course. If unresolved duplicates exist and the student's drops don't resolve them, reject the submission.
2. **Cross-module add/drop duplicate prevention**: Verify/enforce that `validateNoDuplicateCoursesWithCurrentEnrollment()` blocks a student from adding a course in Add/Drop 2 that was already added in Add/Drop 1, and reject duplicate selections in the same submission.
3. **Enable capital validation**: Uncomment/enable the `validateBidPoints()` call in `submitAddDrop()` to prevent negative capital.
4. **Add null-safe financial snapshot handling**: Introduce `getStudentFinancialSnapshot(Student $student)` in `AddDropService` to safely read credits/capital with defaults. Use this snapshot for response calculations and audit logs. Harden `validateBidPoints()` to read capital via null-safe access.
5. **Isolate drop operations by module**: Pass `$moduleId` into `findOneBy()` queries within `AddDropService` and `AddDropValidator` to guarantee drops, responses, and capital refunds strictly affect the active module.
6. **Remove cross-round bidding duplicate prevention**: Remove `validateNoParallelRoundDuplicates()` from `BidValidator` during the Bidding phase. Students are allowed to submit the same bids for the same courses across 2 or more active parallel bidding rounds. The only restrictions during bidding are: (a) cannot bid on previously enrolled courses, and (b) cannot bid on time-conflicting courses.
7. **Fix BidStatus::SUBMITTED reference**: Replace the undefined `BidStatus::SUBMITTED` enum case with `BidStatus::SELECTED` in `BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` to resolve the 500 error on bid submission.
8. **Fix incorrect DQL field `b.moduleId`**: Replace `b.moduleId` with `b.campaignModule` in `BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` to resolve `[Semantical Error]` caused by referencing a non-existent field on the `Bid` entity.
9. **Relax backup selection validation**: Update `BidValidator` to remove conflict and duplicate submission restrictions specifically for backup courses. Allow students to select the same course with different sections in backup and remove time conflict validation for backup courses.
10. **UI disabling for enrolled courses in Add/Drop dropdown**: Update the Add/Drop course selection dropdown in `bidding-web` to preemptively disable courses that the student is already enrolled in. Enrolled courses should appear disabled with a "Previously Enrolled" label, preventing selection at the UI level rather than showing an error after submission.
11. **Add regression tests**: Cover duplicate course resolution, capital validation, null `studentData` behaviors, and the new relaxed backup logic.
12. **Waitlist Duplicate Prevention**: Update `AddDropValidator` to include the `$waitlist` array in all duplicate and previous enrollment checks. Update `BidRepository` to return campaign-wide waitlists, ensuring the UI and backend correctly block courses already waitlisted in other modules.

## Capabilities

### New Capabilities
- `relaxed-backup-validation`: Allow flexible backup selections including duplicate courses in different sections, time conflicts with primary courses, and cross-round duplicates.
- `add-drop-duplicate-course-check`: Detect and reject duplicate courses during Add/Drop, including unresolved duplicate enrollments from parallel bidding, cross-module duplicate prevention, and same-submission duplication.
- `add-drop-studentdata-null-safety`: Add/Drop submission and validation safely handle missing `StudentData` without fatal errors.
- `enforce-capital-validation`: Capital validation is enforced during Add/Drop submissions.
- `ui-blocking-enrolled-courses`: Previously enrolled courses are disabled in the Add/Drop course selection dropdown. They cannot be selected; they show a "Previously Enrolled" reason and are visually disabled.

### Modified Capabilities
- None.

## Impact

- **Affected app(s):** `bidding-api`, `bidding-web`
- **Affected entities:** `Student`, `StudentData`, `AuditLog`, `Bid`
- **Affected domain services:**
  - `App\Domain\Campaign\ActiveCampaign\AddDropService`
  - `App\Domain\Campaign\ActiveCampaign\Validator\AddDropValidator`
  - `App\Domain\Campaign\ActiveCampaign\Validator\BidValidator`
  - `App\Repository\BidRepository`
- **API contract impact:** None. UI blocking relies on existing enrollment state info.
- **Database migration required:** No.
- **Backward compatibility:** Preserved; safely degrades to zero values instead of crashing. Previously-valid submissions with duplicates/negative capital will now be correctly rejected.
- **Affected controllers/components:**
  - `StudentActiveCampaignController`
  - `CreateBiddingAddDropController`
  - `AddDropModal` (frontend)
  - `BiddingCourseList` (frontend)
- **Bidding round behavior change:** Cross-round duplicate prevention is REMOVED from the bidding phase. Students can freely bid on the same course in multiple parallel bidding rounds. Only previously-enrolled course blocking and time conflict validation remain.
