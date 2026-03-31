## Components

### `BidValidator`
- **Path:** `bidding-api/src/Domain/Campaign/ActiveCampaign/Validator/BidValidator.php`
- **Purpose:** Validate student bids during the Bidding phase.
- **Changes:**
  - **Cross-Round Duplicates:** Updated `validateNoParallelRoundDuplicates(...)` to skip validation for bids with `bidType === 'backup'`. This allows students to select courses in backup that they have already submitted/selected in another parallel bidding round.
  - **Submission Duplicates:** Updated `validateNoDuplicates(...)` to allow selecting the same `courseId` multiple times only if they are different `classId`s AND at least one of them is a backup.
  - **Config Key Fix:** Corrected the configuration key from `min_bids_entire_round` to `min_capital_per_student` in `validateCapital()` to align with the actual campaign schema.
  - **Audit Logging:** Integrated `AuditLogService` to log attempts at late submission when the bidding deadline has passed.
  - **Time Conflicts:** Ensure `validateTimeConflicts()` continues to ignore backup bids when evaluating schedule overlaps.

### `BidRepository`
- **Path:** `bidding-api/src/Repository/BidRepository.php`
- **Purpose:** Provide database queries for `Bid` entities.
- **Changes:**
  - Added `findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` to retrieve course IDs that the student has placed bids on in other parallel bidding campaigns within the same program. Used `BidStatus::SELECTED` (not `BidStatus::SUBMITTED`, which does not exist in the enum).
  - Added `findDuplicateEnrolledCoursesByStudentAndCampaign()` to query same-campaign duplicate entries with optional `$moduleId` scoping.
  - Fixed incorrect DQL field reference `b.moduleId` → `b.campaignModule`.

### `AddDropValidator`
- **Path:** `bidding-api/src/Domain/Campaign/ActiveCampaign/Validator/AddDropValidator.php`
- **Purpose:** Validates add/drop and waitlist requests before they are explicitly executed.
- **Changes:**
  - Introduced `validateNoUnresolvedDuplicateEnrollments(Student $student)` to check if the student has multiple ENROLLED/SELECTED bids for the same course in parallel bidding campaigns whose duplicates haven't been resolved with drops.
  - Fixed `validateNoDuplicateCoursesWithCurrentEnrollment` to block adding a course in Add/Drop 2 if the same course is selected in Add/Drop 1, with proper module scoping.

### `AddDropService`
- **Path:** `bidding-api/src/Domain/Campaign/ActiveCampaign/AddDropService.php`
- **Purpose:** Handles the business logic for the Add/Drop phase.
- **Changes:**
  - Added `getStudentFinancialSnapshot(Student $student)` for safe retrieval of points and capital.
  - Uncommented/enabled `validateBidPoints()` during submission.
  - Updated `findOneBy` queries fetching drop bids to filter by the active module ID.

### `bidding-web` (Frontend UI & Validation)
- **Paths:** 
  - `bidding-web/src/features/bidding/utils/validation.util.ts`
  - `bidding-web/src/features/bidding/hooks/use-bidding-form.ts`
  - `bidding-web/src/features/bidding/hooks/use-add-drop-waitlist-form.tsx`
  - `bidding-web/src/features/bidding/hooks/use-course-options.ts`
- **Changes:**
  - **Robust Validation**: Expanded `validateCourseAddition` to accept the full `AvailableCourse` object. It now validates `is_enrolled` status and `unavailable_reason`.
  - **UI Blocking**: The Add Courses dropdown now disables options for previously enrolled courses with a "Previously Enrolled" reason.
  - **Proactive Feedback**: Integrated the updated validation into both `use-bidding-form.ts` and `use-add-drop-waitlist-form.tsx` to show toast notifications if a student tries to add an invalid course.
  - **Stability**: Fixed a critical type mismatch in the form hooks where a credit number was passed to a function expecting a course object.

## State Management

- **Database:** Preserved as is. Logic simply validates and rejects states that breach new duplication/capital rules, or strictly scopes database `UPDATE`s to the correct `active_module_id`.
- **Bid Points (Capital):** Refund computations are isolated to the specific transaction module, guaranteeing users get points refunded strictly for the module they dropped courses in.
- **Errors:** Throw `DomainException` to safely reject bad or duplicate requests across all Add/Drop, Waitlist, and Bidding controllers.

## Internal APIs

- `BidValidator->validate()`: Invoked early during `CreateBiddingService->submit()`. Throw exceptions explicitly highlighting parallel bid duplicate errors for primary bids.
- `AddDropService->submitAddDrop()`: Checks new validation rules early; explicitly maps over drops and adds with module-scoped constraints.
- `AddDropValidator->validateNoUnresolvedDuplicateEnrollments()`: Evaluates pending/unresolved duplicates. Stops execution directly if discrepancies exist.