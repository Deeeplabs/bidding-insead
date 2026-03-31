## Why

Four issues exist across the Bidding and Add/Drop & Waitlist phases:

### 1. Duplicate Course Enrollments from Parallel Bidding Rounds Not Forced to Resolve
In a campaign with parallel bidding, a student can bid on the same course in both rounds. After simulation, both bids can result in ENROLLED status, creating two ENROLLED bids for the same course in the same campaign. The current system has no validation that forces the student to drop one of these duplicates before proceeding with Add/Drop submissions. `validateNoDuplicateCoursesWithCurrentEnrollment()` correctly blocks adding new duplicates, but doesn't force resolution of pre-existing ones.

### 2. Capital (Bid Points) Validation Not Enforced
`validateBidPoints()` exists in `AddDropValidator` but was never called in `AddDropService::submitAddDrop()`. This allowed students to submit add/drop requests that result in negative capital (spending more bid points than available).

### 3. Null-Dereference Failure on missing StudentData
Submitting Add/Drop & Waitlist could fail with a fatal error: `Call to a member function getRemainingCapital() on null`. 
`Student::getStudentData()` is nullable, but key Add/Drop paths dereferenced it directly when building summaries, audit payloads, and validating capital. This blocked submission for students missing `StudentData`.

### 4. Module-Scoped Drop Targeting Flaw
Dropping courses in one parallel bidding round (e.g., `bid2`) erroneously targeted and dropped identical course enrollments from another round (e.g., `bid1`). Drop queries and point refund calculations implicitly fetched the first available bid without scoping it to the active module.

### 5. Duplicate Submission in Parallel Bidding Rounds
Users are able to submit bids for the same exact course across multiple parallel bidding rounds (e.g., BIDDING1 and BIDDING2) during the Bidding phase. There's no cross-round validation to prevent a user from selecting a course in BIDDING2 that they have already submitted/selected in BIDDING1.

## What Changes

1. **Force resolution of parallel bidding duplicates**: Before processing any add/drop submission, detect if the student has multiple ENROLLED/SELECTED bids for the same course. If unresolved duplicates exist and the student's drops don't resolve them, reject the submission.
2. **Cross-module add/drop duplicate prevention**: Verify/enforce that `validateNoDuplicateCoursesWithCurrentEnrollment()` blocks a student from adding a course in Add/Drop 2 that was already added in Add/Drop 1, and reject duplicate selections in the same submission.
3. **Enable capital validation**: Uncomment/enable the `validateBidPoints()` call in `submitAddDrop()` to prevent negative capital.
4. **Add null-safe financial snapshot handling**: Introduce `getStudentFinancialSnapshot(Student $student)` in `AddDropService` to safely read credits/capital with defaults. Use this snapshot for response calculations and audit logs. Harden `validateBidPoints()` to read capital via null-safe access.
5. **Isolate drop operations by module**: Pass `$moduleId` into `findOneBy()` queries within `AddDropService` and `AddDropValidator` to guarantee drops, responses, and capital refunds strictly affect the active module.
6. **Cross-round Bidding duplicate prevention**: Add validation in `BidValidator` during the Bidding phase to retrieve the student's bids in parallel rounds (using a new query in `BidRepository`) and throw an exception if they are trying to bid on a course they already bidded on.
7. **Add regression tests**: Cover duplicate course resolution, capital validation, null `studentData` behaviors, and cross-round Bidding duplicates.

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
