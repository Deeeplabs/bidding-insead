# Fix: Switch — Enable Direct Navigation from Notification Popup to Relevant Page

## Problem
Jira: https://insead.atlassian.net/browse/DPBFAD-925

Programme Managers receive in-app notifications when a student submits or cancels a flex switch request, but the notifications were view-only — there was no link or button to navigate directly to the approval request page. PMs had to manually navigate to `/flex-switch/approval-request` after reading each notification.

Root cause: `FlexSwitchService::submitRequest()` and `FlexSwitchService::cancelRequest()` both call `NotificationService::createBulk()` with the `CUSTOM_ANNOUNCEMENT` type. That template already supports `{{action_url}}` and `{{action_label}}` placeholders, but no values were passed for those keys. `NotificationTemplateResolver` nulls out unresolved placeholders, so `action_url` was always stored as `NULL` on the notification record.

On the frontend, `notification-item.tsx` already conditionally renders a navigation button when `action_url` is present — no frontend changes were needed.

## Changes Made

**`src/Service/FlexSwitch/FlexSwitchService.php`**

- `submitRequest()` — added `'action_url' => '/flex-switch/approval-request'` and `'action_label' => 'View Requests'` to the `$data` array of the PM `createBulk()` call
- `cancelRequest()` — same addition to the PM `createBulk()` call

## Impact

- PM notifications for switch submitted and switch cancelled now include a populated `action_url` and `action_label`, causing `notification-item.tsx` to render a "View Requests" button
- Clicking "View Requests" navigates the PM directly to `/flex-switch/approval-request`
- No impact on student-facing notifications (separate `create()` call, unchanged)
- No impact on other `CUSTOM_ANNOUNCEMENT` usages (action URL is supplied per call, not in the template)
- No database migration, entity changes, or frontend changes required
