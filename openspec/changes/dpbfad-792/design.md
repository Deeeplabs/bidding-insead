## Components

### `BidValidator`
- **Path:** `bidding-api/src/Domain/Campaign/ActiveCampaign/Validator/BidValidator.php`
- **Purpose:** Validate student bids during the Bidding phase.
- **Changes:**
  - **Remove Cross-Round Duplicate Prevention:** Remove `validateNoParallelRoundDuplicates()` call from `validate()`. Students are allowed to submit bids for the same courses across multiple active parallel bidding rounds. This is intentional — cross-round duplicate logic only applies during Add/Drop.
  - **Keep Previously Enrolled Blocking:** `validateNoPreviousEnrollment()` remains unchanged — students cannot bid on courses they are already enrolled in from prior campaigns.
  - **Keep Time Conflict Validation:** `validateTimeConflicts()` remains unchanged for primary bids; backup bids continue to skip conflict checks.
  - **Submission Duplicates:** Updated `validateNoDuplicates(...)` to allow selecting the same `courseId` multiple times only if they are different `classId`s AND at least one of them is a backup.
  - **Config Key Fix:** Corrected the configuration key from `min_bids_entire_round` to `min_capital_per_student` in `validateCapital()` to align with the actual campaign schema.
  - **Audit Logging:** Integrated `AuditLogService` to log attempts at late submission when the bidding deadline has passed.

### `BidRepository`
- **Path:** `bidding-api/src/Repository/BidRepository.php`
- **Purpose:** Provide database queries for `Bid` entities.
- **Changes:**
  - Fixed `findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` — replaced undefined `BidStatus::SUBMITTED` with `BidStatus::SELECTED` and incorrect DQL field `b.moduleId` with `b.campaignModule`. Note: This query is retained for Add/Drop phase duplicate checking only — it is no longer called during Bidding phase submission.
  - Added `findDuplicateEnrolledCoursesByStudentAndCampaign()` to query same-campaign duplicate entries with optional `$moduleId` scoping.

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

### `bidding-web` (Frontend UI — Add/Drop Dropdown Blocking)
- **Paths:** 
  - `bidding-web/src/features/bidding/utils/validation.util.ts`
  - `bidding-web/src/features/bidding/hooks/use-add-drop-waitlist-form.tsx`
  - `bidding-web/src/features/bidding/hooks/use-course-options.ts`
- **Changes:**
  - **Dropdown Disabling**: In the Add/Drop & Waitlist phase, the course selection dropdown (`use-course-options.ts`) preemptively disables courses where `is_enrolled === true`. These courses display a "Previously Enrolled" reason and are visually disabled — students cannot select them.
  - **`getUnavailableReason` priority**: The `is_enrolled` check in `getUnavailableReason()` is applied via `course.unavailable_reason` from the API or the local `is_enrolled` flag. Enrolled courses are disabled before credit checks, conflict checks, or full-class checks.
  - **Robust Validation**: `validateCourseAddition` validates `is_enrolled` status and `unavailable_reason` as a safety net — if a course somehow bypasses dropdown disabling, it is still blocked before being added.
  - **Proactive Feedback**: Integrated validation into `use-add-drop-waitlist-form.tsx` to show toast notifications if a student tries to add an invalid course.
  - **Note**: The bidding round form (`use-bidding-form.ts`) does NOT apply cross-round duplicate prevention at the dropdown level. Students can freely select the same courses across parallel bidding rounds. Only previously-enrolled courses and time conflicts are enforced in the bidding form dropdown.

## State Management

- **Database:** Preserved as is. Logic simply validates and rejects states that breach duplication/capital rules, or strictly scopes database `UPDATE`s to the correct `active_module_id`.
- **Bid Points (Capital):** Refund computations are isolated to the specific transaction module, guaranteeing users get points refunded strictly for the module they dropped courses in.
- **Errors:** Throw `DomainException` to safely reject bad or duplicate requests across all Add/Drop, Waitlist, and Bidding controllers.

## Internal APIs

- `BidValidator->validate()`: Invoked early during `CreateBiddingService->submit()`. Validates previously enrolled courses, time conflicts, backup flexibility, and within-submission duplicates. Does NOT validate cross-round duplicates — students may bid on the same course in multiple parallel rounds.
- `AddDropService->submitAddDrop()`: Checks new validation rules early; explicitly maps over drops and adds with module-scoped constraints.
- `AddDropValidator->validateNoUnresolvedDuplicateEnrollments()`: Evaluates pending/unresolved duplicates. Stops execution directly if discrepancies exist.