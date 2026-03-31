## Why

Seven issues exist across the Bidding and Add/Drop & Waitlist phases:

### 1. Null-Dereference on Missing StudentData
During processing (including summary calculations and audit logging), the API directly dereferenced `Student::getStudentData()` without null checks. For students with missing data, this caused fatal runtime errors (`Call to a member function getRemainingCapital() on null`) blocking Add/Drop submission.

### 2. Unresolved Parallel Bidding Duplicates
In parallel bidding campaigns (multiple modules), students can legitimately end up with duplicate ENROLLED bids for the same course. However, there was no validation forcing them to drop one before performing Add/Drop manipulations. `validateNoDuplicateCoursesWithCurrentEnrollment()` correctly blocks adding new duplicates, but doesn't force resolution of pre-existing ones.

### 3. Capital (Bid Points) Check Disabled
`validateBidPoints()` exists in `AddDropValidator` but was never called in `AddDropService::submitAddDrop()`. This loophole allowed submissions resulting in negative bid point balances.

### 4. Module-Scoped Drop Targeting Flaw
Dropping courses in one parallel bidding round (e.g., `bid2`) erroneously targeted and dropped identical course enrollments from another round (e.g., `bid1`). Drop queries and point refund calculations implicitly fetched the first available bid without scoping it to the active module.

### 5. Direct Duplicate Submission in Parallel Bidding Modules
In a campaign with parallel bidding modules (BIDDING1, BIDDING2), users could independently submit bids for the identical course in both modules. There was no cross-module evaluation of open bids to halt duplicate course selections until after bids were computed to enrollments.

### 6. Undefined BidStatus::SUBMITTED in Cross-Round Query
`BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` referenced `BidStatus::SUBMITTED`, which does not exist in the `BidStatus` enum. PHP throws a fatal `Error` for undefined enum cases before the null-coalescing (`??`) fallback can execute, resulting in a 500 Internal Server Error whenever a student attempts to submit bids during the Bidding phase.

### 7. Incorrect DQL Field Reference `b.moduleId` in Cross-Round Query
`BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` used `b.moduleId` in DQL, but the `Bid` entity has no field or association named `moduleId`. The correct Doctrine association field is `campaignModule` (mapped to column `campaign_module_id`). This caused a `[Semantical Error] line 0, col 200 near 'moduleId != '` on every bid submission when a student had a parallel module to exclude.

## What Changes

1. **Force resolution of parallel bidding duplicates**: Before processing any add/drop submission, detect if the student has multiple ENROLLED/SELECTED bids for the same course. If unresolved duplicates exist and the student's drops don't resolve them, reject the submission.
2. **Cross-module add/drop duplicate prevention**: Verify/enforce that `validateNoDuplicateCoursesWithCurrentEnrollment()` blocks a student from adding a course in Add/Drop 2 that was already added in Add/Drop 1, and reject duplicate selections in the same submission.
3. **Enable capital validation**: Uncomment/enable the `validateBidPoints()` call in `submitAddDrop()` to prevent negative capital.
4. **Add null-safe financial snapshot handling**: Introduce `getStudentFinancialSnapshot(Student $student)` in `AddDropService` to safely read credits/capital with defaults. Use this snapshot for response calculations and audit logs. Harden `validateBidPoints()` to read capital via null-safe access.
5. **Isolate drop operations by module**: Pass `$moduleId` into `findOneBy()` queries within `AddDropService` and `AddDropValidator` to guarantee drops, responses, and capital refunds strictly affect the active module.
6. **Cross-round Bidding duplicate prevention**: Add validation in `BidValidator` during the Bidding phase to retrieve the student's bids in parallel rounds (using a new query in `BidRepository`) and throw an exception if they are trying to bid on a course they already bidded on.
7. **Fix BidStatus::SUBMITTED reference**: Replace the undefined `BidStatus::SUBMITTED` enum case with `BidStatus::SELECTED` in `BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` to resolve the 500 error on bid submission.
8. **Fix incorrect DQL field `b.moduleId`**: Replace `b.moduleId` with `b.campaignModule` in `BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` to resolve `[Semantical Error]` caused by referencing a non-existent field on the `Bid` entity.
9. **Add regression tests**: Cover duplicate course resolution, capital validation, null `studentData` behaviors, and cross-round Bidding duplicates.

## Capabilities

### New Capabilities
- `bidding-duplicate-course-prevent`: Prevent users from bidding on the same course across parallel bidding rounds during the Bidding phase.
- `add-drop-duplicate-course-check`: Detect and reject duplicate courses during Add/Drop, including unresolved duplicate enrollments from parallel bidding, cross-module duplicate prevention, and same-submission duplication.
- `add-drop-studentdata-null-safety`: Add/Drop submission and validation safely handle missing `StudentData` without fatal errors.
- `enforce-capital-validation`: Capital validation is enforced during Add/Drop submissions.

### Modified Capabilities
- None.

## Impact

- **Affected app(s):** `bidding-api`
- **Affected entities:** `Student`, `StudentData`, `AuditLog`
- **Affected domain services:**
  - `App\Domain\Campaign\ActiveCampaign\AddDropService`
  - `App\Domain\Campaign\ActiveCampaign\Validator\AddDropValidator`
  - `App\Domain\Campaign\ActiveCampaign\Validator\BidValidator`
  - `App\Repository\BidRepository`
- **API contract impact:** None. Validation errors use existing `\DomainException` pattern.
- **Database migration required:** No.
- **Backward compatibility:** Preserved; safely degrades to zero values instead of crashing. Previously-valid submissions with duplicates/negative capital will now be correctly rejected.
- **Affected controllers:**
  - `StudentActiveCampaignController`
  - `CreateBiddingAddDropController`
  - `CampusExchangeAddDropWaitlistController`
  - `CampusExchangeAddDropAllotmentController`
