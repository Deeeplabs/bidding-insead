## 1. Verify/Create Notification Type

- [x] 1.1 Check if an existing NotificationType or System Alert type exists for "SIS Sync Failure". If not, create a Doctrine Migration to insert a new NotificationType.
- [x] 1.2 Create or execute `bin/console doctrine:migrations:migrate` if a migration was created.

## 2. Implement Notification Trigger

- [x] 2.1 Identify the exact locations where SIS sync (PeopleSoft) failures occur (e.g., in `bidding-api/src/Controller/Api/Webhook/` controllers or `bidding-api/src/Service/Import/` services).
- [x] 2.2 Inject `NotificationService` into the identified webhook controllers or import services.
- [x] 2.3 Implement the error handling logic to trigger `NotificationService->createNotification(...)` when a sync failure payload or exception is caught.
- [x] 2.4 Set the notification message to precisely: "Student or course data synchronization with the SIS has failed. Please investigate."

## 3. Recipient Resolution

- [x] 3.1 Update the trigger logic to query users who hold the `ROLE_BUSINESS_PARTNER` or `ROLE_PROGRAM_MANAGER` roles.
- [x] 3.2 Ensure the notification is dispatched to the identified users.

## 4. Testing

- [x] 4.1 Write a PHPUnit test in `bidding-api/tests/Unit/Controller/Api/Webhook/` (or the relevant domain test directory) to mock a sync failure and verify that `NotificationService->createNotification` is called with the correct message and roles.
- [x] 4.2 Manually test by triggering a mock SIS failure via cURL/Postman or console command, then verify in the admin database that the notification entry was created.
- [x] 4.3 Verify in the `bidding-admin` frontend (Notification Centre) that the notification displays correctly for a logged-in BP or PM user.
