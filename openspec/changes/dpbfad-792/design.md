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

### `StudentActiveCampaignController`
- **Path:** `bidding-api/src/Controller/Api/Student/Campaign/StudentActiveCampaignController.php`
- **Purpose:** Serves the `/student/active-campaigns/{id}/bidding/{moduleId}/available` endpoint used by the bidding dropdown.
- **Changes:**
  - **Fix `is_enrolled` flag for parallel bidding rounds**: In `getAvailableCourses()`, the `$allEnrolledCourseIds` query (line 681-683) currently calls `findEnrolledCourseIdsByStudentAndProgram($student, $program)` **without** excluding the current campaign. This causes courses with `SELECTED` bids in the same campaign's other modules (e.g., bidding round 1) to have `is_enrolled = true` in the API response. The frontend then disables them as "Previously Enrolled" in bidding round 2. **Fix**: Pass `$campaign` as the third argument to also exclude the current campaign — aligning `$allEnrolledCourseIds` with `$previouslyEnrolledCourseIds`. This ensures only courses from **prior campaigns** are flagged as enrolled, while courses from parallel bidding rounds in the same campaign remain selectable.

### `BidRepository`
- **Path:** `bidding-api/src/Repository/BidRepository.php`
- **Purpose:** Provide database queries for `Bid` entities.
- **Changes:**
  - Fixed `findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` — replaced undefined `BidStatus::SUBMITTED` with `BidStatus::SELECTED` and incorrect DQL field `b.moduleId` with `b.campaignModule`. Note: This query is retained for Add/Drop phase duplicate checking only — it is no longer called during Bidding phase submission.
  - Added `findDuplicateEnrolledCoursesByStudentAndCampaign()` to query same-campaign duplicate entries with optional `$moduleId` scoping. **Waitlist Fix**: Removed module scoping for waitlist status, making it campaign-wide to ensure cross-module waitlist duplicates are detected.
  - **`findEnrolledOrWaitlistedCourseIdsByStudentAndCampaign`**: Broadened to always return campaign-wide waitlists, even if a `$moduleId` is provided. This ensures the UI and backend correctly identify courses waitlisted in other phases as duplicates.

### `AddDropValidator`
- **Path:** `bidding-api/src/Domain/Campaign/ActiveCampaign/Validator/AddDropValidator.php`
- **Purpose:** Validates add/drop and waitlist requests before they are explicitly executed.
- **Changes:**
  - Introduced `validateNoUnresolvedDuplicateEnrollments(Student $student)` to check if the student has multiple ENROLLED/SELECTED bids for the same course in parallel bidding campaigns whose duplicates haven't been resolved with drops.
  - **Waitlist inclusion in duplicate checks**: `validateNoDuplicateCoursesInSubmission`, `validateNoPreviousEnrollment`, and `validateNoDuplicateCoursesWithCurrentEnrollment` now accept and validate the `$waitlist` array. This prevents students from waitlisting for courses they are already enrolled in or have already waitlisted for in other modules.
  - **`validateNoDuplicateCoursesWithCurrentEnrollment` — module-scoped drop exclusion fix**: The function now accepts an additional `?int $moduleId` parameter. When computing `$droppedCourseIds` (courses to exclude from the duplicate check), it ONLY excludes a course if the student has an ENROLLED or SELECTED bid for that class **in the current module** (`campaignModule = $moduleId`). This prevents a student from bypassing the cross-phase duplicate check by including a class from Add/Drop 1 in their Add/Drop 2 `drops` list.
  - `validateDrops` is NOT changed to be module-scoped — this remains intentionally campaign-wide so the validator correctly confirms the student has a droppable bid (regardless of which module it belongs to). The module scoping for actual drop execution remains in `AddDropService::executeCampaignAddDrop` (which filters by `campaignModule = $moduleId` when dropping). The duplicate-bypass gap is closed at the `validateNoDuplicateCoursesWithCurrentEnrollment` level, not at `validateDrops`.

### `AddDropService`
- **Path:** `bidding-api/src/Domain/Campaign/ActiveCampaign/AddDropService.php`
- **Purpose:** Handles the business logic for the Add/Drop phase.
- **Changes:**
  - Added `getStudentFinancialSnapshot(Student $student)` for safe retrieval of points and capital.
  - Uncommented/enabled `validateBidPoints()` during submission.
  - Updated `findOneBy` queries fetching drop bids to filter by the active module ID.
  - **Waitlist Validation**: Updated all validator calls to pass the `$waitlist` array. Ensured validators are called even if only waitlist items are present in the submission.
  - **Pass `$moduleId` to `validateNoDuplicateCoursesWithCurrentEnrollment`** in Step 5c so that the drop exclusion inside that function is module-scoped. Previously this call passed no `$moduleId`, which allowed the cross-module drop bypass.

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
- `AddDropService->submitAddDrop()`: Checks new validation rules early; explicitly maps over drops and adds with module-scoped constraints. Now passes `$moduleId` to `validateNoDuplicateCoursesWithCurrentEnrollment` at Step 5c.
- `AddDropValidator->validateNoUnresolvedDuplicateEnrollments()`: Evaluates pending/unresolved duplicates from parallel bidding rounds (scoped to current module). Stops execution directly if discrepancies exist.
- `AddDropValidator->validateNoDuplicateCoursesWithCurrentEnrollment($enrollments, $drops, $student, $campaign, $moduleId)`: Now accepts `$moduleId`. When building `$droppedCourseIds`, queries the database to confirm each dropped class has an enrolled bid in the current module. Only those confirmed drops exclude their course from the campaign-wide duplicate map. This ensures a student cannot use a cross-module class in their drop list to bypass the cross-add/drop-phase duplicate check.