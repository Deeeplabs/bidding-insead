# Fix: Flex Switch Audit Logs Not Appearing on Audit Logs Page

## Problem

After adding `AuditLogService` calls to all FlexSwitch mutation operations (approve, reject, submit, cancel), audit log entries were **still not visible** on the Monitoring & Analytics > Audit Logs page.

Reported endpoints affected:
- `POST /v2/api/student/flex-switch/request` — student submits a flex switch request
- `POST /v2/api/dashboard/flex-switch/approval-requests/{id}/process` — PM approves/rejects a request

> **Note:** `GET /v2/api/dashboard/flex-switch/approval-requests?page=1&limit=5` is a read-only list endpoint — it does NOT create audit logs by design.

### Root Cause

The `AuditLogService->log()` calls used the default `async: true`, which dispatches `AuditLogMessage` to the Symfony Messenger `async` transport (`config/packages/messenger.yaml`). The transport DSN depends on the `MESSENGER_TRANSPORT_DSN` env var — if this is a queue-based transport (Doctrine, Redis, AMQP), a running consumer worker (`php bin/console messenger:consume async`) is required. If the worker is not running or the transport is misconfigured, messages pile up in the queue but never persist to the `audit_log` table.

Additionally, both `AuditLogService::log()` and `AuditLogMessageHandler::__invoke()` catch ALL exceptions silently (logging to file only), making errors invisible to users and API responses.

## Solution

Switched all FlexSwitch audit log calls to **synchronous mode** (`async: false`), bypassing the Messenger bus entirely. This writes audit entries directly to the `audit_log` table in the same request via `AuditLogRepository::save()`.

### Changes Made

**Modified File:**

1. **`src/Service/FlexSwitchApprovalService.php`**
   - Updated `auditLogService->log()` call in `processApproval()` — added `async: false`
   - Logs `APPROVE FlexSwitchRequest` or `REJECT FlexSwitchRequest` action synchronously after flush

2. **`src/Service/FlexSwitch/FlexSwitchService.php`**
   - Updated `auditLogService->log()` call in `submitRequest()` — added `async: false`
   - Logs `CREATE FlexSwitchRequest` action synchronously after save
   - Updated `auditLogService->log()` call in `cancelRequest()` — added `async: false`
   - Logs `CANCEL FlexSwitchRequest` action synchronously after save

## Audit Log Actions Summary

| Operation                 | Action                         | Entity Type          | Data Captured                                                  |
|---------------------------|--------------------------------|----------------------|----------------------------------------------------------------|
| PM approves request       | `APPROVE FlexSwitchRequest`    | `FlexSwitchRequest`  | Status transition, approver, student/course info, remarks      |
| PM rejects request        | `REJECT FlexSwitchRequest`     | `FlexSwitchRequest`  | Status transition, approver, student/course info, remarks      |
| Student submits request   | `CREATE FlexSwitchRequest`     | `FlexSwitchRequest`  | From/to class/course IDs, reason, student info                 |
| Student cancels request   | `CANCEL FlexSwitchRequest`     | `FlexSwitchRequest`  | Status transition, cancellation reason, student info           |

## Impact

- **Audit Compliance:** All FlexSwitch state-changing operations now reliably persist to the `audit_log` table
- **No API Changes:** Purely backend change — no response shape modifications
- **No Migrations:** Uses existing `audit_log` table infrastructure
- **No Breaking Changes:** All existing FlexSwitch operations continue to function identically
- **Synchronous Logging:** Adds a small DB write per mutation (~1-5ms). Acceptable for low-frequency FlexSwitch operations

## Verification

- [ ] Query DB: `SELECT * FROM audit_log WHERE entity_type = 'FlexSwitchRequest' ORDER BY created_at DESC LIMIT 10;`
- [ ] PM approval/rejection creates audit log entries with correct action, entity type, old/new data
- [ ] Student submit creates audit log entry with course details
- [ ] Student cancel creates audit log entry with status transition
- [ ] Existing audit log entries (configuration, course adjustment) remain unaffected
- [ ] All new entries visible in Monitoring & Analytics > Audit Logs page
