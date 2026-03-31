## Context

PM users receive in-app notifications when a student submits or cancels a flex switch request. These notifications are created via `FlexSwitchService::submitRequest()` and `cancelRequest()` using the `CUSTOM_ANNOUNCEMENT` notification type. The `CUSTOM_ANNOUNCEMENT` template already defines `{{action_url}}` and `{{action_label}}` placeholders — when these are unresolved (no matching key in `$data`), `NotificationTemplateResolver` nulls them out. As a result, the `action_url` column on stored `Notification` records is `NULL`.

On the frontend, `notification-item.tsx` already renders an action button when `action_url` is present. No frontend work is needed.

## Goals / Non-Goals

**Goals:**
- PM notifications for switch submitted and switch cancelled include a populated `action_url` (`/flex-switch/approval-request`) and `action_label` (`View Requests`)
- PM can click "View Requests" from the notification to navigate directly to the approval page

**Non-Goals:**
- Deep-linking to a specific request (e.g. `/flex-switch/approval-request?id=123`) — the tech note specifies `/flex-switch/approval-request` as the target
- Changing the student-facing notification for their own submission (that notification is sent via a separate `create()` call and is not part of this ticket)
- Modifying the `FlexSwitchApprovalService` PM notifications (separate notification path, out of scope)

## Decisions

**Decision: Pass `action_url` and `action_label` via the `$data` array rather than updating the database template**

The `CUSTOM_ANNOUNCEMENT` template already resolves `{{action_url}}` from `$data`. Adding the two keys to existing `createBulk()` calls is a single-file change with zero risk of affecting other notification types that also use `CUSTOM_ANNOUNCEMENT`. Updating the database template itself would change the URL for every `CUSTOM_ANNOUNCEMENT` notification system-wide (reminders, announcements, etc.), which would be incorrect.

Alternative considered: Create a new `FLEX_SWITCH_PM` notification type with a hardcoded action URL in its template. Rejected — adds unnecessary complexity (new fixture, migration, type registration) for a change that can be solved with two lines per call site.

## Risks / Trade-offs

- [Risk] If the admin route `/flex-switch/approval-request` changes in the future, the hardcoded URL will break silently (notifications will link to a 404).
  → Mitigation: the URL is a stable admin page path; document it in the spec as a contract.

- [Risk] Existing stored notifications (already in the database) will remain without `action_url`. PMs will only see the button on new notifications created after deployment.
  → Acceptable: retroactive backfill is not required.
