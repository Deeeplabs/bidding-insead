# Fix Switch Notification — PM Not Notified on Student Cancel

## Problem

When a student cancels a flex switch request via `PUT /student/flex-switch/my-requests/{id}/cancel`, Programme Manager (PM) users do not receive any notification about the cancellation. PMs already receive notifications when a student **submits** a switch request (implemented in `FlexSwitchService::submitRequest()`), but the `cancelRequest()` method was missing the same notification logic. This means PMs have no visibility into cancelled requests unless they manually check the approval dashboard.

## Solution

Added PM notification dispatch to `FlexSwitchService::cancelRequest()`, following the exact same pattern already used in `submitRequest()`.

### Changes Made

**Modified File:**

1. **`src/Service/FlexSwitch/FlexSwitchService.php`**
   - **Method**: `cancelRequest(int $requestId, Student $student, string $cancellationReason)`
   - Added notification trigger after successful cancellation save (after `$this->requestRepository->save()`)
   - Resolves the student's Promotion → Program → ProgramManagers chain to find PM recipients
   - Resolves from/to course names from the request's class IDs
   - Sends a `CUSTOM_ANNOUNCEMENT` bulk notification to all PM users via `NotificationService::createBulk()` with:
     - **Title**: "Flex Switch Request Cancelled"
     - **Body**: "Student {name} has cancelled their switch request from {fromCourse} to {toCourse}. Reason: {reason}."
     - **Data**: `request_id`, `student_id`, `from_course`, `to_course`, `cancellation_reason`
   - Gracefully handles edge cases: no promotion, no program, no PMs configured — cancellation still succeeds without errors

## Impact

- **No API response shape changes**: The `PUT /student/flex-switch/my-requests/{id}/cancel` response is unchanged
- **No frontend changes needed**: Notification appears in existing PM notification feed
- **No migration required**: No database schema changes
- **No new dependencies**: Uses existing `NotificationService` and `UserRepository` already injected into the service
- **Backward compatible**: Purely additive side-effect — cancellation behavior is unchanged

## Verification

- [ ] Call `PUT /v2/api/student/flex-switch/my-requests/{id}/cancel` with a valid cancellation reason on a pending request
- [ ] Verify the request is cancelled successfully (status 200)
- [ ] Check the `notification` table for new PM notification records with title "Flex Switch Request Cancelled"
- [ ] Verify notification body contains correct student name, course names, and cancellation reason
- [ ] Verify no notification error when student has no promotion/program or no PMs are configured
