# System Notifications for SIS Sync Failures

## Problems

### 1. Lack of Visibility for SIS Sync Failures
Currently, when a synchronization mechanism with the SIS (PeopleSoft) fails for any student or course data, there is no in-app alert or notification notifying administrators. This lack of visibility can lead to data drift or synchronization inconsistencies going unnoticed by Business Partners and Program Managers, requiring late manual intervention or confusing student portals.

**Root cause**: Webhooks handling SIS integrations do not trigger internal notifications upon error.

## Solution

### 1. Add SIS Sync Failure Notification
Implement an immediate notification triggered upon a PeopleSoft (SIS) synchronization error. This ensures system anomalies are surfaced directly to Business Partners and Program Managers for immediate investigation.

## Changes Made

1. **`src/Repository/UserRepository.php`**
   - Added `findBusinessPartnersAndProgramManagers()` method to accurately query for all users holding the `ROLE_BUSINESS_PARTNER` or `ROLE_PROGRAM_MANAGER` roles.

2. **Webhook Controllers**
   - Modified the following controllers:
     - `CreateCourseController`
     - `CreateClassController`
     - `CreateStudentController`
     - `DeleteStudentController`
     - `DeleteClassController`
     - `EnrollmentController`
   - Wrapped the main logic in `try-catch` blocks or modified existing error handling.
   - Injected `NotificationService` and `UserRepository`.
   - Triggered `NotificationService->createBulk(...)` using the pre-existing `SIS_SYNC_FAILED` notification type when integration faults are detected.
   - The notification message is configured via the pre-existing `SIS_SYNC_FAILED` template ("Student or course data synchronization with the SIS has failed. Please investigate.").

3. **`src/Domain/Notification/NotificationService.php`**
   - Removed the `final` keyword from the class declaration to allow it to be mocked in unit tests.

## Impact

- **bidding-api**: Additions to Webhook failure handlers. Introduces a query for BP/PM users to dispatch notifications.
- **bidding-admin**: BP/PM users will now begin receiving this notification natively within their Notification Centre. No UI code changes are strictly necessary aside from relying on existing display behavior.
- **Backward compatible**: Will not conflict with existing notifications. Non-disruptive to data handling. Resolves a critical "silent failure" state.

## Verification

- [x] `bidding-api/tests/Unit/Controller/Api/Webhook/Course/Create/CreateCourseControllerTest.php` — Created a PHPUnit test to trigger an intentional SIS sync failure by throwing an exception in the mock `CourseService`, verifying that `NotificationService->createBulk` receives the `SIS_SYNC_FAILED` code and the resolved array of BP/PM users.
- [x] Manual triggered failure via Postman/cURL correctly fires database `Notification` insertion.
- [x] Visual verification inside `bidding-admin` Notification Centre shows the alert to an active PM/BP assigned user.
