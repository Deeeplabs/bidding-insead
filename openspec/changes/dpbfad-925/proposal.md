## Why

Programme Managers receive notifications when students submit or cancel flex switch requests, but the notifications are view-only — there is no link to navigate directly to the approval request page. PMs must manually navigate to `/flex-switch/approval-request` after reading the notification, which adds friction to their workflow.

Despite `action_url` and `action_label` values already being passed in the `$data` array of the `createBulk()` calls, the `action_label` field on the stored `Notification` entity is still `null`. The root cause is in `NotificationTemplateResolver::render()` — its fallback logic uses strict null comparison (`=== null`) to decide whether to use `$data['action_label']`. If the `CUSTOM_ANNOUNCEMENT` template's `actionLabel` column in the database is an empty string (`''`) rather than `NULL` (e.g. a PM edited the template via the admin UI and cleared the field), the fallback is skipped and the empty string propagates to the notification.

## What Changes

- **Root-cause fix** in `NotificationTemplateResolver::render()`: change the fallback guard from `=== null` to a falsy check (`empty()`) for both `actionUrl` and `actionLabel`, so that `null`, `''`, and unresolved placeholders all trigger the data-array fallback.
- **Data-passing** (already in place): `FlexSwitchService::submitRequest()` and `cancelRequest()` PM `createBulk()` calls pass `'action_url' => '/flex-switch/approval-request'` and `'action_label' => 'View Requests'` in `$data`.
- The `notification-item.tsx` frontend component already conditionally renders a navigation button when `action_url` is present — no frontend changes are required.

## Capabilities

### New Capabilities
- `switch-notification-action-url`: Populate `action_url` and `action_label` on PM flex switch notifications so the notification item renders a "View Requests" button linking to `/flex-switch/approval-request`

### Modified Capabilities
_(none — no existing spec-level requirements are changing)_

## Impact

- **Domain**: `bidding-api/src/Domain/Notification/NotificationTemplateResolver.php`
  - `render()` — fix fallback guard for `actionUrl` and `actionLabel` from `=== null` to falsy check
- **Service**: `bidding-api/src/Service/FlexSwitch/FlexSwitchService.php`
  - `submitRequest()` — PM `createBulk` call: `'action_url'` and `'action_label'` already present in `$data` (no change needed)
  - `cancelRequest()` — PM `createBulk` call: `'action_url'` and `'action_label'` already present in `$data` (no change needed)
- **API**: No response shape changes
- **Frontend**: No changes required — `notification-item.tsx` already renders the action button when `action_url` is present
- **Migration**: None required — no schema changes
- **Breaking changes**: None
