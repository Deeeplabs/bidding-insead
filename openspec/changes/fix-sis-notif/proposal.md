## Why

The system needs to alert Business Partners and Program Managers (PM) when PeopleSoft (SIS) synchronization fails so they can immediately investigate and resolve the issue.

## What Changes

- Introduce a new System Notification for "PeopleSoft (SIS) Sync Failure".
- Trigger this notification immediately upon a SIS integration error.
- Send the notification to users with the Business Partner and Program Manager (PM) roles.
- The notification message will be: "Student or course data synchronization with the SIS has failed. Please investigate."

## Capabilities

### New Capabilities
- `system-notifications`: Implements system-level alerts (such as SIS integration errors) directed to Business Partners and Program Managers.

### Modified Capabilities

## Impact

- **bidding-api**: Modification of the MuleSoft/webhook or SIS synchronization service to trigger the new notification upon failure.
- **Entities**: Potentially impacts the `NotificationType` or `NotificationSetting` to include the new system alert category.
- **bidding-admin**: No major UI changes besides ensuring the new notification type is displayed gracefully in the notification-centre.
