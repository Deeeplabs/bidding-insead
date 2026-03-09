## Context

Currently, when a student submits a flex switch request (`FlexSwitchService::submitRequest`), PM users are notified via `NotificationService::createBulk()`. The notification logic finds PMs through `Student → Promotion → Program → ProgramManager → User`. However, the `FlexSwitchService::cancelRequest()` method has no notification logic — PMs are not informed when a student cancels their request.

Both `NotificationService` and `UserRepository` are already injected into `FlexSwitchService`. The `ProgramManager` entity repository is already queried in `submitRequest`. No new infrastructure is needed.

## Goals / Non-Goals

**Goals:**
- Notify PM users when a student cancels a flex switch request
- Follow the exact same notification pattern used in `submitRequest()` for consistency
- Include student name, course details, and cancellation reason in the notification

**Non-Goals:**
- Adding new notification types (we'll reuse `CUSTOM_ANNOUNCEMENT`)
- Changing the cancel API response shape
- Adding email notifications (uses existing in-app notification channel)
- Notifying the student themselves (they initiated the cancellation)

## Decisions

1. **Reuse `CUSTOM_ANNOUNCEMENT` type** — Same as `submitRequest`. No new notification type registration needed.
2. **Same PM lookup pattern** — `Student → Promotion → Program → ProgramManager → User`. Identical to `submitRequest` lines 807-835.
3. **Notification placement** — After saving cancellation and history entry, before returning result. This mirrors the submit flow.
4. **Notification content** — Title: "Flex Switch Request Cancelled", Body: "Student {name} has cancelled their switch request from {fromCourse} to {toCourse}. Reason: {reason}."
5. **No notification to the student** — The student performed the action themselves; no confirmation notification needed.

## Risks / Trade-offs

- **Low risk**: This is purely additive — a side-effect after a successful operation. No existing behavior changes.
- **Notification failure handling**: If notification dispatch fails, the cancellation has already been persisted. This is the same behavior as `submitRequest` — notification failures are non-blocking. The `NotificationService::createBulk` already handles errors gracefully.
- **No PMs found**: If the student has no promotion, no program, or no PMs are configured, no notification is sent. This is consistent with `submitRequest` behavior.
