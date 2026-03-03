# Fix Edit Switch — Validation, API Enrichment & Notifications

## Problem

The Flex Switch feature had several gaps across filtering, validation, rule enforcement, and communication:

1. **Incomplete Filtering**: The `/v2/api/flex-switch-class-configuration` endpoint did not strictly filter by Programme, Promotion, or Core course type (`courseType.id = 6`), allowing non-Core or unrelated courses to appear in the configuration list.
2. **Missing Response Fields**: The same endpoint's response payload lacked `module` and `term` details required by the frontend for proper display.
3. **No Enrollment Validation**: Students could submit a flex switch request targeting a course they were already enrolled in, leading to duplicate enrollments.
4. **No Schedule Conflict Detection**: There was no check against the student's active enrolled classes for scheduling conflicts before allowing a switch.
5. **Unenforced Rule Configuration**: Bid points rules (`deduct_points`, `minimum_points`) and `max_submissions_allowed_per_student` from `FlexSwitchConfiguration` were not evaluated during flex switch submissions.
6. **No Notifications**: Students received no notification when submitting a flex switch request or when a Programme Manager approved/rejected their request. Notification template variables `{{announcement_title}}` and `{{announcement_body}}` were passed as raw placeholders instead of resolved content.
7. **Limited Search Functionality**: The `/student/flex-switch/switch-to-courses` endpoint only supported searching by course name, course ID, and section — lacking the ability to search by module or home campus.
8. **Campus Filtering**: The `/student/flex-switch/switch-to-courses` endpoint allowed students to select switch target classes that were located at their own campus, which contradicted the business logic of flex switches.

## Solution

Centralized all flex switch validations into the domain service layer, enriched the API response with promotion period metadata, enforced PM-configured rules, integrated notification triggers with properly resolved content, and enhanced search capabilities.

### Changes Made

**Modified Files:**

1. **`src/Service/FlexSwitch/PM/FlexSwitchService.php`**
   - Updated `listClassConfiguration()` query to enforce `courseType.id = 6` (Core) filtering.
   - Added `program_id` and `promotion_id` filter support via `cp.promotion` and `pFilter.program` joins.
   - Added `MAX(pp.period) as module` and `MAX(pp.term) as term` to the SELECT clause, joining `classPromotions → promotionPeriod`.
   - Mapped `module` and `term` values into `FlexSwitchClassConfigurationItemDto` in the result transformer loop.

2. **`src/Domain/FlexSwitch/FlexSwitchClassConfigurationItemDto.php`**
   - Added nullable `$module` (string) and `$term` (string) properties with OpenAPI annotations. Additive change — backward compatible.

3. **`src/Controller/Api/FlexSwitch/FlexSwitchController.php`**
   - Passes `program_id` and `promotion_id` from query parameters into the `listClassConfiguration()` filter array.

4. **`src/Service/FlexSwitch/FlexSwitchService.php`**
   - Implemented `submitRequest()` with full domain validation pipeline:
     - **Already Enrolled check**: Queries `Bid` entity for enrolled status on the target course (skipped for same-course section switches). Throws `\DomainException` on duplicate.
     - **Schedule Conflict check**: Queries active enrolled bids (excluding the from-class), compares `classConflicts` arrays bidirectionally between target and enrolled classes. Throws `\DomainException` with conflicting course name.
     - **Bid Points Rule check**: Loads `FlexSwitchConfiguration` by student's promotion, evaluates `rules.deduct_points` and `rules.minimum_points` against student's remaining capital. Throws `\DomainException` if insufficient.
     - **Max Submissions check**: Counts existing flex switch submissions for the student and rejects if `max_submissions_allowed_per_student` limit is reached.
   - Injected `NotificationService` and `UserRepository` via constructor.
   - After successful request creation, triggers a `CUSTOM_ANNOUNCEMENT` notification to the student with resolved `announcement_title` ("Flex Switch Request Submitted") and `announcement_body` (includes actual from/to course names).
   - Triggers an additional `CUSTOM_ANNOUNCEMENT` bulk notification to all Programme Managers associated with the student's program to alert them of the new flex switch request.
   - Enhanced `getSwitchToCourses()` with expanded search functionality supporting:
     - Course name (existing)
     - Course ID (existing)
     - Section (existing)
     - Module name (NEW - e.g., "Module 1", "Module 2")
     - Campus name (NEW - e.g., "Singapore", "France")
     - Campus code (NEW - e.g., "SGP", "FRA")

5. **`src/Controller/Api/Student/FlexSwitch/FlexSwitchController.php`**
   - Delegates all validation to `flexSwitchService->submitRequest()`.
   - Maps `\DomainException` to HTTP 400 JSON responses.
   - Updated OpenAPI documentation for `/student/flex-switch/switch-to-courses` to reflect enhanced search parameter description.

6. **`src/Service/FlexSwitchApprovalService.php`**
   - Injected `NotificationService` via constructor.
   - After `processApproval()` completes, triggers a `CUSTOM_ANNOUNCEMENT` notification to the student with resolved `announcement_title` ("Flex Switch Request Approved/Rejected") and `announcement_body` (includes from/to course names, action status, and PM remarks).

## Impact

- **API Response Enrichment**: `/v2/api/flex-switch-class-configuration` now returns `module` and `term` fields per class configuration item. No existing fields removed — purely additive.
- **Stricter Filtering**: Only Core courses matching the student's Programme and Promotion are returned, preventing invalid switch targets.
- **Domain Validation**: Enrollment duplicates, schedule conflicts, bid point thresholds, and submission limits are all enforced at the service layer before request creation.
- **Notification Coverage**: Students and Programme Managers receive immediate notifications on flex switch request submission and on approval/rejection processing, with fully resolved content (no raw template placeholders).
- **Enhanced Search**: The `search` query parameter on `/student/flex-switch/switch-to-courses` now supports searching by module name and campus (name or code), in addition to existing course name, course ID, and section searches. Example: `?from_class_id=19142&search=module`
