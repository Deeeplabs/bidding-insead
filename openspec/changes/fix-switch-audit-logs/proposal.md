## Why

When a Program Manager (PM) interacts with the Flex Switch module — specifically approving/rejecting student switch requests — none of these interactions are recorded in the system Audit Logs. Similarly, when students submit or cancel flex switch requests, these actions are also missing from audit trails.

This is a compliance and traceability gap. The audit log system exists and is actively used for other modules (user management, campaign phases, simulations, course adjustments, etc.), but the FlexSwitch approval workflow was never wired to it.

**Previous fix attempt**: `AuditLogService->log()` calls were added to `FlexSwitchApprovalService.processApproval()`, `FlexSwitchService.submitRequest()`, and `FlexSwitchService.cancelRequest()`. However, audit log entries are **still not visible** on the Audit Logs menu page. The same issue affects the existing audit log calls in `PM\FlexSwitchService.saveCourseAdjustment()` and `createConfiguration()`.

**Root cause analysis** (revised):
1. **Async message delivery** — ALL FlexSwitch audit log calls (across 3 services, 10 total calls) use the default `async: true`, which dispatches `AuditLogMessage` to the Symfony Messenger `async` transport (`config/packages/messenger.yaml` line 17). The transport DSN comes from `MESSENGER_TRANSPORT_DSN` env var. If this is a queue-based transport (Doctrine, Redis, AMQP), messages require a running consumer worker (`php bin/console messenger:consume async`). If the worker isn't running or the transport is misconfigured, audit messages queue up but never persist to the `audit_log` table.
2. **Silent error swallowing** — Both `AuditLogService::log()` and `AuditLogMessageHandler::__invoke()` catch ALL exceptions silently (only logging to application log file). Any serialization, dispatch, or persistence errors are invisible to the user and API response.
3. **Endpoint clarification** — `GET /v2/api/dashboard/flex-switch/approval-requests?page=1&limit=5` is a **read endpoint** that lists requests — no audit log. The actual approval action uses `POST .../approval-requests/{id}/process`. Similarly, `GET /v2/api/flex-switch/course-adjustment/{course_id}/{class_id}` is a read endpoint — no audit log. The mutation is via `POST /v2/api/flex-switch/course-adjustment/{course_id}/{class_id}`.

## What Changes

Two-part fix — ensure audit log entries are reliably persisted AND visible:

### Part A: Reliable Audit Log Persistence
1. **Switch FlexSwitch audit log calls to synchronous mode** (`async: false`) — Bypass the message bus entirely. This ensures audit entries are written to the `audit_log` table in the same request, matching the synchronous fallback path in `AuditLogService::log()`. This eliminates the async transport dependency.
   - `FlexSwitchApprovalService.processApproval()` — Update `log()` call with `async: false` (1 call)
   - `FlexSwitchService.submitRequest()` — Update `log()` call with `async: false` (1 call)
   - `FlexSwitchService.cancelRequest()` — Update `log()` call with `async: false` (1 call)
   - `PM\FlexSwitchService.createConfiguration()` — Update `log()` calls with `async: false` (2 calls: CREATE/UPDATE FlexSwitchConfiguration)
   - `PM\FlexSwitchService.saveCourseAdjustment()` — Update `log()` calls with `async: false` (5 calls: UPDATE Course, UPDATE Class, UPDATE ClassPromotion, CREATE/UPDATE FlexCourseAdjustment)

### Part B: Verification & Diagnostics
2. **Verify entries exist in `audit_log` table** — Query the database directly after performing operations to confirm rows are created with correct `entity_type` values (`FlexSwitchRequest`, `Course`, `Class`, `ClassPromotion`, `FlexCourseAdjustment`, `FlexSwitchConfiguration`).
3. **Check application logs for silent errors** — Search for `"Failed to save audit log"` error messages in the Symfony log files.

## Capabilities

### New Capabilities
- `flex-switch-audit-logging`: Ensure reliable audit log persistence for all FlexSwitch mutation operations — approval processing (approve/reject), student request lifecycle (submit/cancel), PM course adjustment, and PM configuration

### Modified Capabilities
<!-- No existing spec-level behavior changes — this adds audit logging to existing operations -->

## Impact

- **API (bidding-api)**:
  - `src/Service/FlexSwitchApprovalService.php` — Update `AuditLogService->log()` call in `processApproval()` to use `async: false` (1 call)
  - `src/Service/FlexSwitch/FlexSwitchService.php` — Update `AuditLogService->log()` calls in `submitRequest()` and `cancelRequest()` to use `async: false` (2 calls)
  - `src/Service/FlexSwitch/PM/FlexSwitchService.php` — Update `AuditLogService->log()` calls in `createConfiguration()` and `saveCourseAdjustment()` to use `async: false` (7 calls)
- **No API response changes** — Audit logging is a side-effect, no response shape changes
- **No migration required** — Uses existing `audit_log` table
- **No breaking changes** — Purely additive behavior
- **Performance note** — Synchronous logging adds a small DB write to each request. This is acceptable for the low-frequency FlexSwitch operations.
