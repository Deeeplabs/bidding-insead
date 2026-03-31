## Context

PM users receive in-app notifications when a student submits or cancels a flex switch request. These notifications are created via `FlexSwitchService::submitRequest()` and `cancelRequest()` using the `CUSTOM_ANNOUNCEMENT` notification type and `NotificationService::createBulk()`.

The `CUSTOM_ANNOUNCEMENT` template fixture defines `actionLabel = '{{action_label}}'` and `actionUrlTemplate = '{{action_url}}'`. When `createBulk()` is called, the template resolver (`NotificationTemplateResolver::render()`) renders the template and resolves placeholders from the `$data` array. The `$data` array already contains `'action_url' => '/flex-switch/approval-request'` and `'action_label' => 'View Requests'` (added in a prior iteration). However, `action_label` is still `null` on stored notifications.

### Root cause analysis

The bug is in `NotificationTemplateResolver::render()` (lines 72–86). The fallback logic that decides whether to use `$data['action_label']` vs the template-rendered value uses a **strict null check**:

```php
$actionLabel = $template->renderActionLabel($data);

if ($actionLabel === null || str_contains($actionLabel, '{{')) {
    $actionLabel = $data['action_label'] ?? $actionLabel;
}
```

**Failure scenario:** If the `CUSTOM_ANNOUNCEMENT` template's `actionLabel` column is an **empty string** (`''`) in the database — which happens when a PM edits the template via `PUT /notification-templates/{id}` and sends `"action_label": ""` (or the field is saved as blank) — then:

1. `$template->renderActionLabel($data)` → calls `replacePlaceholders('', $data)` → returns `''`
2. `'' === null` → **false** → fallback is **skipped**
3. `str_contains('', '{{')` → **false** → fallback is **skipped**
4. `$actionLabel` remains `''`
5. Notification stored with `actionLabel = ''`
6. Frontend treats `''` as falsy → no button rendered

The same issue applies to `actionUrl`.

On the frontend, `notification-item.tsx` already renders an action button when `action_url` is present. No frontend work is needed.

## Goals / Non-Goals

**Goals:**
- Fix the `NotificationTemplateResolver::render()` fallback guard so that empty strings also trigger the `$data` fallback
- PM notifications for switch submitted and switch cancelled include a populated `action_url` (`/flex-switch/approval-request`) and `action_label` (`View Requests`)
- PM can click "View Requests" from the notification to navigate directly to the approval page

**Non-Goals:**
- Deep-linking to a specific request (e.g. `/flex-switch/approval-request?id=123`) — the tech note specifies `/flex-switch/approval-request` as the target
- Changing the student-facing notification for their own submission (that notification is sent via a separate `create()` call and is not part of this ticket)
- Modifying the `FlexSwitchApprovalService` PM notifications (separate notification path, out of scope)

## Decisions

**Decision 1: Fix the resolver fallback guard (primary fix)**

Change the fallback condition from strict null to a falsy check so that `null`, `''`, and unresolved placeholders all trigger the data-array fallback:

```php
// Before (broken for empty string):
if ($actionLabel === null || str_contains($actionLabel, '{{')) {

// After (handles null, empty string, and unresolved placeholders):
if (empty($actionLabel) || str_contains($actionLabel, '{{')) {
```

Apply the same fix for `actionUrl`. This is a two-line change in `NotificationTemplateResolver::render()` that fixes the root cause for ALL notification types, not just flex switch.

**Decision 2: Keep `action_url` and `action_label` in `$data` (already in place)**

The `createBulk()` calls in `submitRequest()` and `cancelRequest()` already pass `'action_url' => '/flex-switch/approval-request'` and `'action_label' => 'View Requests'` in `$data`. No further changes needed in `FlexSwitchService`.

Alternative considered: Create a new `FLEX_SWITCH_PM` notification type with a hardcoded action URL in its template. Rejected — adds unnecessary complexity (new fixture, migration, type registration) for a change that can be solved with two lines in the resolver plus existing data passing.

## Risks / Trade-offs

- [Risk] The `empty()` check in the resolver changes behavior for all notification types: any template with `actionLabel = ''` will now fall back to `$data['action_label']` instead of storing `''`.
  → Acceptable: an empty-string action label has no useful meaning; falling back to data is the correct behavior.

- [Risk] If the admin route `/flex-switch/approval-request` changes in the future, the hardcoded URL will break silently (notifications will link to a 404).
  → Mitigation: the URL is a stable admin page path; document it in the spec as a contract.

- [Risk] Existing stored notifications (already in the database) will remain without `action_url`. PMs will only see the button on new notifications created after deployment.
  → Acceptable: retroactive backfill is not required.
