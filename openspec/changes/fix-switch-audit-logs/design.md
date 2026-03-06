## Context

The FlexSwitch module allows students to request course switches, and PMs to approve/reject those requests. The system has a comprehensive `AuditLogService` used across many modules (user management, campaigns, simulations, course adjustments). However, audit logging coverage in FlexSwitch is incomplete:

**Currently audited** (in `PM\FlexSwitchService`):
- `createConfiguration()` — logs CREATE/UPDATE FlexSwitchConfiguration ✅
- `saveCourseAdjustment()` — logs Course, Class, ClassPromotion, and FlexCourseAdjustment changes ✅

**NOT audited** (the gap this change fixes):
- `FlexSwitchApprovalService.processApproval()` — approve/reject requests ❌
- `FlexSwitchService.submitRequest()` — student submits a switch request ❌
- `FlexSwitchService.cancelRequest()` — student cancels a pending request ❌

## Goals / Non-Goals

**Goals:**
- Add audit log entries for all FlexSwitch state-changing operations that currently lack them
- Follow the exact same `AuditLogService->log()` pattern already used in `PM\FlexSwitchService`
- Record meaningful old/new data snapshots for each operation
- No changes to API response shapes or database schema

**Non-Goals:**
- Changing the audit log entity or table structure
- Adding audit logging for read operations (list, get)
- Adding frontend UI changes for viewing these new audit entries (they'll appear in the existing Audit Logs page automatically)
- Modifying the existing audit log entries in `PM\FlexSwitchService`

## Decisions

### 1. Inject `AuditLogService` into services that lack it
- `FlexSwitchApprovalService` — needs new constructor dependency on `AuditLogService`
- `FlexSwitchService` (student side) — needs new constructor dependency on `AuditLogService`

### 2. Audit log entity types and actions

| Operation | Entity Type | Action | Old Data | New Data |
|-----------|-------------|--------|----------|----------|
| PM approves request | `FlexSwitchRequest` | `APPROVE FlexSwitchRequest` | status: pending | status: approved, approver, remarks |
| PM rejects request | `FlexSwitchRequest` | `REJECT FlexSwitchRequest` | status: pending | status: rejected, approver, remarks |
| Student submits request | `FlexSwitchRequest` | `CREATE FlexSwitchRequest` | null | request details (from/to class, reason) |
| Student cancels request | `FlexSwitchRequest` | `CANCEL FlexSwitchRequest` | status: pending | status: cancelled, cancellation reason |

### 3. Place audit calls after successful flush
Following the existing pattern in `PM\FlexSwitchService`, audit log calls should be placed after `entityManager->flush()` to ensure we only log successfully persisted operations.

### 4. Use async logging (default)
The `AuditLogService->log()` method defaults to async via message bus. We'll use this default to avoid impacting request latency.

## Risks / Trade-offs

- **Low risk**: Purely additive change — no response shapes, database schema, or existing behavior changes
- **Message bus dependency**: Audit logs are dispatched asynchronously. If the message bus consumer is down, logs will be queued but not lost
- **Student-side logging**: The `FlexSwitchService` (student side) does not currently have `Security` injected, but `AuditLogService` handles user resolution internally via `Security` — so the student's user context will be captured automatically from the request
