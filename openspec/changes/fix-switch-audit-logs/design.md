## Context

The FlexSwitch module allows students to request course switches, and PMs to approve/reject those requests. The system has a comprehensive `AuditLogService` used across many modules (user management, campaigns, simulations, course adjustments). However, audit logging coverage in FlexSwitch is incomplete:

**Audit log calls exist but use async** (all affected by the same root cause):
- `PM\FlexSwitchService.createConfiguration()` — logs CREATE/UPDATE FlexSwitchConfiguration (2 calls, async)
- `PM\FlexSwitchService.saveCourseAdjustment()` — logs Course, Class, ClassPromotion, FlexCourseAdjustment changes (5 calls, async)
- `FlexSwitchApprovalService.processApproval()` — logs APPROVE/REJECT FlexSwitchRequest (1 call, async)
- `FlexSwitchService.submitRequest()` — logs CREATE FlexSwitchRequest (1 call, async)
- `FlexSwitchService.cancelRequest()` — logs CANCEL FlexSwitchRequest (1 call, async)

**Total: 10 audit log calls across 3 services — all using default `async: true`.**

**Previous attempt status**: Audit log calls were added to `FlexSwitchApprovalService` and student-side `FlexSwitchService`. The `PM\FlexSwitchService` already had audit log calls. However, entries are NOT appearing in the Audit Logs page for ANY of these. Investigation revealed the root cause is the **async message delivery mechanism**.

**Key endpoint clarification**:
- `POST /v2/api/student/flex-switch/request` → calls `submitRequest()` → SHOULD create audit log
- `POST /v2/api/dashboard/flex-switch/approval-requests/{id}/process` → calls `processApproval()` → SHOULD create audit log
- `POST /v2/api/flex-switch/course-adjustment/{course_id}/{class_id}` → calls `saveCourseAdjustment()` → SHOULD create audit logs (up to 5 per call)
- `GET /v2/api/dashboard/flex-switch/approval-requests?page=1&limit=5` → READ-only → no audit log
- `GET /v2/api/flex-switch/course-adjustment/{course_id}/{class_id}` → READ-only → no audit log

## Goals / Non-Goals

**Goals:**
- Ensure audit log entries for ALL FlexSwitch state-changing operations are **reliably persisted** to the `audit_log` table
- Fix the async delivery issue by switching to synchronous logging for all 10 FlexSwitch audit log calls across 3 services
- Verify entries are visible on the Monitoring & Analytics > Audit Logs page
- No changes to API response shapes or database schema

**Non-Goals:**
- Changing the audit log entity or table structure
- Adding audit logging for read operations (list, get endpoints)
- Adding frontend UI changes for viewing these new audit entries (they appear in the existing Audit Logs page automatically)
- Fixing the global async messenger transport configuration (out of scope — sync override is the targeted fix)

## Decisions

### 1. AuditLogService already injected in all services
- `FlexSwitchApprovalService` — has `AuditLogService` constructor dependency ✅
- `FlexSwitchService` (student side) — has `AuditLogService` constructor dependency ✅
- `PM\FlexSwitchService` — has `AuditLogService` constructor dependency ✅

### 2. Audit log entity types and actions (full scope)

| Operation | Entity Type | Action | Service Method |
|-----------|-------------|--------|----------------|
| PM approves request | `FlexSwitchRequest` | `APPROVE FlexSwitchRequest` | `FlexSwitchApprovalService.processApproval()` |
| PM rejects request | `FlexSwitchRequest` | `REJECT FlexSwitchRequest` | `FlexSwitchApprovalService.processApproval()` |
| Student submits request | `FlexSwitchRequest` | `CREATE FlexSwitchRequest` | `FlexSwitchService.submitRequest()` |
| Student cancels request | `FlexSwitchRequest` | `CANCEL FlexSwitchRequest` | `FlexSwitchService.cancelRequest()` |
| PM creates config | `FlexSwitchConfiguration` | `CREATE FlexSwitchConfiguration` | `PM\FlexSwitchService.createConfiguration()` |
| PM updates config | `FlexSwitchConfiguration` | `UPDATE FlexSwitchConfiguration` | `PM\FlexSwitchService.createConfiguration()` |
| PM updates course | `Course` | `UPDATE Course` | `PM\FlexSwitchService.saveCourseAdjustment()` |
| PM updates class | `Class` | `UPDATE Class` | `PM\FlexSwitchService.saveCourseAdjustment()` |
| PM updates class promotion | `ClassPromotion` | `UPDATE ClassPromotion` | `PM\FlexSwitchService.saveCourseAdjustment()` |
| PM creates adjustment | `FlexCourseAdjustment` | `CREATE FlexCourseAdjustment` | `PM\FlexSwitchService.saveCourseAdjustment()` |
| PM updates adjustment | `FlexCourseAdjustment` | `UPDATE FlexCourseAdjustment` | `PM\FlexSwitchService.saveCourseAdjustment()` |

### 3. Place audit calls after successful flush (unchanged)
Following the existing pattern in `PM\FlexSwitchService`, audit log calls are placed after `entityManager->flush()` to ensure we only log successfully persisted operations.

### 4. Use SYNCHRONOUS logging (`async: false`) — REVISED
The previous implementation used the default `async: true`, which dispatches `AuditLogMessage` to the Symfony Messenger `async` transport (`config/packages/messenger.yaml`). This transport depends on `MESSENGER_TRANSPORT_DSN` env var and requires a running worker (`php bin/console messenger:consume async`).

**Problem**: If the transport is queue-based (Doctrine, Redis, AMQP) and the worker isn't running or encounters errors, messages are queued but never persisted to the `audit_log` table. Both `AuditLogService::log()` and `AuditLogMessageHandler::__invoke()` catch ALL exceptions silently — errors are only written to application log files, invisible to users.

**Fix**: Pass `async: false` to all FlexSwitch `AuditLogService->log()` calls. This triggers the synchronous fallback path in `AuditLogService::log()` (lines 112-132), which directly creates and saves the `AuditLog` entity via `AuditLogRepository::save()`. This bypasses the message bus entirely.

**Trade-off**: Adds a small DB write to each FlexSwitch mutation request. Acceptable for the low-frequency of these operations (individual student requests and PM approvals).

### 5. Verification approach
After applying the fix:
- Query `audit_log` table directly: `SELECT * FROM audit_log WHERE entity_type IN ('FlexSwitchRequest', 'Course', 'Class', 'ClassPromotion', 'FlexCourseAdjustment', 'FlexSwitchConfiguration') ORDER BY created_at DESC`
- Perform test operations (submit, approve, cancel, course adjustment) and confirm entries appear
- Check the Audit Logs page in the UI (Monitoring & Analytics > Audit Logs)

## Risks / Trade-offs

- **Low risk**: Only changes the `async` parameter on existing audit log calls — no logic changes
- **Synchronous DB write**: Adds ~1-5ms per FlexSwitch mutation. Negligible for low-frequency operations
- **Student-side user context**: `AuditLogService` resolves the user from `Security::getUser()` automatically. For student-authenticated requests, this captures the student's user context
- **Silent error handling**: If the synchronous write still fails, the exception is caught by the try/catch in `AuditLogService::log()`. Check application logs for `"Failed to save audit log"` messages if entries still don't appear
