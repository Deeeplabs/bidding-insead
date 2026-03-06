## Why

When a Program Manager (PM) interacts with the Flex Switch module — specifically approving/rejecting student switch requests — none of these interactions are recorded in the system Audit Logs. Similarly, when students submit or cancel flex switch requests, these actions are also missing from audit trails.

This is a compliance and traceability gap. The audit log system exists and is actively used for other modules (user management, campaign phases, simulations, course adjustments, etc.), but the FlexSwitch approval workflow was never wired to it.

**Previous fix attempt**: `AuditLogService->log()` calls were added to `FlexSwitchApprovalService.processApproval()`, `FlexSwitchService.submitRequest()`, and `FlexSwitchService.cancelRequest()`. However, audit log entries are **still not visible** on the Audit Logs menu page.

**Root cause analysis** (revised):
1. **Async message delivery** — `AuditLogMessage` is routed to the `async` Messenger transport (`config/packages/messenger.yaml` line 17). The transport DSN comes from `MESSENGER_TRANSPORT_DSN` env var. If this is a queue-based transport (Doctrine, Redis, AMQP), messages require a running consumer worker (`php bin/console messenger:consume async`). If the worker isn't running or the transport is misconfigured, audit messages queue up but never persist to the `audit_log` table.
2. **Silent error swallowing** — Both `AuditLogService::log()` and `AuditLogMessageHandler::__invoke()` catch ALL exceptions silently (only logging to application log file). Any serialization, dispatch, or persistence errors are invisible to the user and API response.
3. **Endpoint clarification** — The user-reported endpoint `/v2/api/dashboard/flex-switch/approval-requests?page=1&limit=5` is a **GET (read) endpoint** that lists requests. It does NOT trigger mutations or audit logs. The actual approval action uses `POST /v2/api/dashboard/flex-switch/approval-requests/{id}/process`, which does trigger audit logging.

## What Changes

Two-part fix — ensure audit log entries are reliably persisted AND visible:

### Part A: Reliable Audit Log Persistence
1. **Switch FlexSwitch audit log calls to synchronous mode** (`async: false`) — Bypass the message bus entirely. This ensures audit entries are written to the `audit_log` table in the same request, matching the synchronous fallback path in `AuditLogService::log()`. This eliminates the async transport dependency.
   - `FlexSwitchApprovalService.processApproval()` — Update `log()` call with `async: false`
   - `FlexSwitchService.submitRequest()` — Update `log()` call with `async: false`
   - `FlexSwitchService.cancelRequest()` — Update `log()` call with `async: false`

### Part B: Verification & Diagnostics
2. **Verify entries exist in `audit_log` table** — Query the database directly after performing operations to confirm rows are created with correct `entity_type = 'FlexSwitchRequest'`.
3. **Check application logs for silent errors** — Search for `"Failed to save audit log"` error messages in the Symfony log files.

## Capabilities

### New Capabilities
- `flex-switch-audit-logging`: Add audit log entries for FlexSwitch approval processing (approve/reject) and student request lifecycle (submit/cancel)

### Modified Capabilities
<!-- No existing spec-level behavior changes — this adds audit logging to existing operations -->

## Impact

- **API (bidding-api)**:
  - `src/Service/FlexSwitchApprovalService.php` — Update `AuditLogService->log()` call in `processApproval()` to use `async: false`
  - `src/Service/FlexSwitch/FlexSwitchService.php` — Update `AuditLogService->log()` calls in `submitRequest()` and `cancelRequest()` to use `async: false`
- **No API response changes** — Audit logging is a side-effect, no response shape changes
- **No migration required** — Uses existing `audit_log` table
- **No breaking changes** — Purely additive behavior
- **Performance note** — Synchronous logging adds a small DB write to each request. This is acceptable for the low-frequency FlexSwitch operations.
