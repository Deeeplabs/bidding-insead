# Fix: Switch — Enable Direct Navigation from Notification Popup to Relevant Page

## Problem
Jira: https://insead.atlassian.net/browse/DPBFAD-925

Programme Managers receive in-app notifications when a student submits or cancels a flex switch request, but the notifications were view-only — there was no link or button to navigate directly to the approval request page. PMs had to manually navigate to `/flex-switch/approval-request` after reading each notification.

**Root cause:** The `NotificationTemplateResolver::render()` method uses a strict null comparison (`=== null`) in its fallback guard for `actionUrl` and `actionLabel`. When the `CUSTOM_ANNOUNCEMENT` template's `actionLabel` column in the database is an empty string (`''`) rather than `NULL` (e.g. a PM edited the template via the admin UI and cleared the field), the fallback to `$data['action_label']` is skipped — the empty string passes the `=== null` check and the `str_contains('{{')` check, so it propagates to the notification unchanged. The frontend treats `''` as falsy, so no action button is rendered.

The `$data` array in `FlexSwitchService::submitRequest()` and `cancelRequest()` already included `'action_url'` and `'action_label'` values (added in a prior iteration), but they were never reaching the stored notification due to this resolver bug.

On the frontend, `notification-item.tsx` already conditionally renders a navigation button when `action_url` is present — no frontend changes were needed.

## Changes Made

**`src/Domain/Notification/NotificationTemplateResolver.php`**

- `render()` — changed the `actionUrl` fallback guard from `$actionUrl === null` to `empty($actionUrl)` so that both `null` and `''` trigger the `$data['action_url']` fallback
- `render()` — changed the `actionLabel` fallback guard from `$actionLabel === null` to `empty($actionLabel)` so that both `null` and `''` trigger the `$data['action_label']` fallback

**`src/Service/FlexSwitch/FlexSwitchService.php`** (no change — already in place)

- `submitRequest()` — PM `createBulk()` call already includes `'action_url' => '/flex-switch/approval-request'` and `'action_label' => 'View Requests'` in `$data`
- `cancelRequest()` — same data already present in the PM `createBulk()` call

## Impact

- Fixes the root cause in `NotificationTemplateResolver` for all notification types — any template with an empty-string `actionLabel` or `actionUrl` will now correctly fall back to `$data` values
- PM notifications for switch submitted and switch cancelled now include a populated `action_url` and `action_label`, causing `notification-item.tsx` to render a "View Requests" button
- Clicking "View Requests" navigates the PM directly to `/flex-switch/approval-request`
- No impact on student-facing notifications (separate `create()` call, unchanged)
- No impact on other `CUSTOM_ANNOUNCEMENT` usages where `action_url`/`action_label` are not provided in `$data` (they remain `null` as before)
- No database migration, entity changes, or frontend changes required
