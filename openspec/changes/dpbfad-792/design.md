## Context

The bidding system Add/Drop & Waitlist phase has several missing guardrails and a runtime crash vulnerability:
1. **Parallel Pre-bidding Duplicates**: Students with duplicate ENROLLED bids for the same course from separate parallel bidding rounds were not forced to drop one before submitting Add/Drop requests.
2. **Capital Validation Disabled**: `validateBidPoints()` was implemented but commented out, allowing submissions to result in negative capital.
3. **Missing Null Safety**: `Student::getStudentData()` is nullable, but was dereferenced directly in Add/Drop summary calculation, audit payload generation, and bid point validation, causing `Call to a member function getRemainingCapital() on null`.
4. **Duplicate Safeguards**: Cross-module and same-submission duplicate add/drop validations were present but lacked formal coverage and artifact documentation.
5. **Drop Targeting Flaws**: Dropping courses in `bid2` would erroneously drop identical course enrollments belonging to `bid1`.

## Goals / Non-Goals

**Goals:**
- Prevent Add/Drop progression if unresolved duplicate course enrollments exist from parallel bidding.
- Enable capital (bid points) validation to block negative capital submissions.
- Eliminate null-dereference failures by centralizing a financial fallback helper (`getStudentFinancialSnapshot`) and using null-safe operators.
- Ensure cross-module duplicate detection and drop-then-add logic works as intended and is covered by tests.
- Isolate drop operations and capital refund queries to the active module (`$moduleId`) so parallel bidding rounds don't intersect.
- Keep duplicate-course guardrail behavior explicit in regression tests.

**Non-Goals:**
- Preventing parallel bidding rounds from generating duplicate enrollments initially (allowed by design).
- Modifying database schema or API response shapes.
- Refactoring `studentData` dereferencing outside of Add/Drop hot paths.

## Decisions

### 1. Unresolved Duplicate Detection (`AddDropValidator` & `BidRepository`)
- Add `findDuplicateEnrolledCoursesByStudentAndCampaign(Student $student, Campaign $campaign, ?int $moduleId = null)` to `BidRepository`. This query groups ENROLLED/SELECTED bids by `course_id` and returns those with `COUNT > 1`. When `$moduleId` is provided, only checks duplicates within that specific module.
- Add `validateNoUnresolvedDuplicateEnrollments(array $drops, Student $student, Campaign $campaign, ?int $moduleId = null)` to `AddDropValidator`. It checks if any identified duplicates remain unresolved after applying the current request's drops. Throws `\DomainException` if so. When `$moduleId` is provided, only validates within that module's scope.
- This runs *before* other enrollments checks in `AddDropService::submitAddDrop()`.
- **Critical Fix**: The validation now correctly scopes to the current bidding round (bid1 or bid2) using `$moduleId`, preventing Add/Drop in bid2 from seeing duplicates from bid1.

### 2. Enable Capital Validation
- Uncomment `$this->validator->validateBidPoints(...)` in `AddDropService::submitAddDrop()`. It runs after `validateCreditLimits()` and is guarded by `!empty($enrollments)`.

### 3. Null-Safe Financial Fallback
- Add `getStudentFinancialSnapshot(Student $student): array{credits_taken: float, remaining_capital: int}` to `AddDropService`.
- Defaults to `credits_taken = 0.0` and `remaining_capital = 0` if `studentData` is null.
- Use this snapshot in `buildResponse()` and `createAuditLog()`.
- Update `validateBidPoints()` to use `$student->getStudentData()?->getRemainingCapital() ?? 0`.

### 4. Module-Scoped Add/Drop Filtering
- Pass `$moduleId` into `buildResponse()`, `validateBidPoints()`, and drop tracking.
- Apply `['campaignModule' => $moduleId]` to `findOneBy()` array limits. This precisely locks operations to the round the student interacts with, resolving cross-module drop bugs.

### 5. Lock in Existing Duplicate Guardrails
- Verify and test existing `validateNoDuplicateCoursesInSubmission` (same-submission duplicates) and `validateNoDuplicateCoursesWithCurrentEnrollment` (campaign-scoped, cross-module duplicates).
- Ensure drop-then-add of the same course in one request is correctly permitted.

## Risks / Trade-offs

- **Strictness Trade-off**: Enabling capital validation and enforcing duplicate course resolution will block previously allowed invalid flows. This is desired but might surprise users.
- **Fallback Semantics**: Missing `studentData` gracefully degrading to zero prevents crashes but might obscure underlying data issues. A follow-up to harden hotspots or validate data integrity may be useful.
- **Performance**: One additional `GROUP BY` database query per submission, which is negligible.
