## Components

### `BidValidator`
- **Path:** `bidding-api/src/Domain/Campaign/ActiveCampaign/Validator/BidValidator.php`
- **Purpose:** Validate student bids during the Bidding phase.
- **Changes:**
  - **Cross-Round Duplicates:** Update `validateNoParallelRoundDuplicates(...)` to skip validation for bids with `bidType === 'backup'`. This allows students to select courses in backup that they have already submitted/selected in another parallel bidding round.
  - **Submission Duplicates:** Update `validateNoDuplicates(...)` to allow selecting the same `courseId` multiple times only if they are different `classId`s AND at least one of them is a backup.
  - **Time Conflicts:** (Already handled) Ensure `validateTimeConflicts()` continues to ignore backup bids when evaluating schedule overlaps.

### `BidRepository`
- **Path:** `bidding-api/src/Repository/BidRepository.php`
- **Purpose:** Provide database queries for `Bid` entities.
- **Changes:**
  - Add `findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` to retrieve course IDs that the student has placed bids on in other parallel bidding campaigns within the same program. Use `BidStatus::SELECTED` (not `BidStatus::SUBMITTED`, which does not exist in the enum) to filter for bids in the submitted/active state during the Bidding phase.
  - Add `findDuplicateEnrolledCoursesByStudentAndCampaign()` to query same-campaign duplicate entries with optional `$moduleId` scoping.
  - Fix incorrect DQL field reference `b.moduleId` → `b.campaignModule` in the exclude-module clause of `findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()`. The `Bid` entity has no `moduleId` field; the correct association is `campaignModule`.

### `AddDropValidator`
- **Path:** `bidding-api/src/Domain/Campaign/ActiveCampaign/Validator/AddDropValidator.php`
- **Purpose:** Validates add/drop and waitlist requests before they are explicitly executed.
- **Changes:**
  - Introduce `validateNoUnresolvedDuplicateEnrollments(Student $student)` to check if the student has multiple ENROLLED/SELECTED bids for the same course in parallel bidding campaigns whose duplicates haven't been resolved with drops.
  - Fix `validateNoDuplicateCoursesWithCurrentEnrollment` to block adding a course in Add/Drop 2 if the same course is selected in Add/Drop 1 but the module ID query didn't restrict them previously. Scope it better.

### `AddDropService`
- **Path:** `bidding-api/src/Domain/Campaign/ActiveCampaign/AddDropService.php`
- **Purpose:** Handles the business logic for the Add/Drop phase, including applying drops, additions, point refunds, and logging.
- **Changes:**
  - Add `getStudentFinancialSnapshot(Student $student)` for safe retrieval of points and capital.
  - Uncomment/enable `validateBidPoints()` during submission.
  - Update `findOneBy` queries fetching drop bids so they explicitly accept and filter by the active module ID (`$moduleId`), preventing cross-module drop accidents.

### `bidding-web` (Frontend UI Blocking)
- **Path:** `bidding-web/src/components/AddDrop/AddDropModal.tsx` (and related list components)
- **Purpose:** Manage the course selection interface for students.
- **Changes:**
  - Update the course selection list to check against the student's existing enrollments (already provided via initial state/API).
  - Disable or hide "Add" buttons/checkboxes for courses the student is already enrolled in for this program.
  - Provide a clear tooltip or visual indicator (e.g. "Already Enrolled") instead of allowing interaction and failing later with a toast error.

## State Management

- **Database:** Preserved as is. Logic simply validates and rejects states that breach new duplication/capital rules, or strictly scopes database `UPDATE`s to the correct `active_module_id`.
- **Bid Points (Capital):** Refund computations are isolated to the specific transaction module, guaranteeing users get points refunded strictly for the module they dropped courses in.
- **Errors:** Throw `DomainException` to safely reject bad or duplicate requests across all Add/Drop, Waitlist, and Bidding controllers.

## Internal APIs

- `BidValidator->validate()`: Invoked early during `CreateBiddingService->submit()`. Throw exceptions explicitly highlighting parallel bid duplicate errors for primary bids.
- `AddDropService->submitAddDrop()`: Checks new validation rules early; explicitly maps over drops and adds with module-scoped constraints.
- `AddDropValidator->validateNoUnresolvedDuplicateEnrollments()`: Evaluates pending/unresolved duplicates. Stops execution directly if discrepancies exist.