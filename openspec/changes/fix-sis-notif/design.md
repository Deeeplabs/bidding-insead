## Context

The INSEAD Bidding System relies on integrations with external systems, specifically PeopleSoft (SIS), for student and course data. Current synchronization tasks are handled largely via MuleSoft webhooks (`/api/webhook/{class,course,enrollment,student}/*`) and other background sync processes. When an integration error occurs, there is currently no systematic push notification informing the relevant administrative staff—Business Partners and Program Managers (PM)—that data drift may be happening due to a sync failure.

## Goals / Non-Goals

**Goals:**
- Create a mechanism to trigger a notification when a SIS integration/sync error occurs.
- Route this notification specifically to users holding the Business Partner and Program Manager roles.
- Ensure the notification includes the message: "Student or course data synchronization with the SIS has failed. Please investigate."

**Non-Goals:**
- Changing the actual sync mechanics or fixing underlying SIS/MuleSoft integration bugs.
- Sending notifications via external channels (e.g. email/SMS) if the system only supports in-app notifications (we will utilize existing `NotificationService` behavior).

## Decisions

- **Leverage Existing Notification Framework:** The system already has a robust Notification framework (`Notification`, `NotificationBatch`, `NotificationType`). We will introduce a new `NotificationType` (or reuse an appropriate system alert type if one exists) explicitly for SIS Sync Failure.
- **Trigger Location:** The trigger for this notification will be placed in the webhook controllers and/or data import services (`bidding-api/src/Controller/Api/Webhook/`, `bidding-api/src/Service/Import/`) wherever a failure corresponding to SIS sync is caught.
- **Recipient Resolution:** The `NotificationService` will be queried to find all users with the `ROLE_BUSINESS_PARTNER` or `ROLE_PROGRAM_MANAGER` (or their database equivalents in the Role entity) and attach them as recipients.

## Risks / Trade-offs

- **[Risk] Notification Spam:** If a sync failure occurs repeatedly (e.g., a batch process failing for an hour), it could create hundreds of notifications for every PM/BP. 
  → **Mitigation:** If applicable we should restrict the notification to trigger once per batch or throttle/debounce it using cache, or simply rely on the fact that existing webhook error handlers might already batch their feedback. For this implementation, we will follow standard simple notification triggers first.
- **[Risk] Role resolution performance:** Fetching all users globally with a BP/PM role to send them individual notifications could be slow if there are thousands of PMs.
  → **Mitigation:** The active PM/BP user pool is constrained to staff, making the query footprint small.
