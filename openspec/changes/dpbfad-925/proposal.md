## Why

Programme Managers receive notifications when students submit or cancel flex switch requests, but the notifications are view-only — there is no link to navigate directly to the approval request page. PMs must manually navigate to `/flex-switch/approval-request` after reading the notification, which adds friction to their workflow.

## What Changes

- Add `action_url` and `action_label` to the `$data` array passed to `notificationService->createBulk()` in `FlexSwitchService::submitRequest()` (PM notification block) and `FlexSwitchService::cancelRequest()` (PM notification block).
- The `CUSTOM_ANNOUNCEMENT` template already supports `{{action_url}}` and `{{action_label}}` placeholders; they are currently unresolved because no values are passed for them, causing the template resolver to null them out. Passing the values will populate the `action_url` and `action_label` fields on stored `Notification` entities.
- The `notification-item.tsx` frontend component already conditionally renders a navigation button when `action_url` is present — no frontend changes are required.

## Capabilities

### New Capabilities
- `switch-notification-action-url`: Populate `action_url` and `action_label` on PM flex switch notifications so the notification item renders a "View Requests" button linking to `/flex-switch/approval-request`

### Modified Capabilities
_(none — no existing spec-level requirements are changing)_

## Impact

- **Service**: `bidding-api/src/Service/FlexSwitch/FlexSwitchService.php`
  - `submitRequest()` — PM `createBulk` call: add `'action_url' => '/flex-switch/approval-request'` and `'action_label' => 'View Requests'` to `$data`
  - `cancelRequest()` — PM `createBulk` call: add `'action_url' => '/flex-switch/approval-request'` and `'action_label' => 'View Requests'` to `$data`
- **API**: No response shape changes
- **Frontend**: No changes required — `notification-item.tsx` already renders the action button when `action_url` is present
- **Migration**: None required — no schema changes
- **Breaking changes**: None
