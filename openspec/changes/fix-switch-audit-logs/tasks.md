## 1. Switch Audit Log Calls to Synchronous Mode (Root Cause Fix)

The existing audit log calls use the default `async: true`, which dispatches `AuditLogMessage` to the Symfony Messenger `async` transport. Messages are queued but may not be consumed if the worker is down or the transport is misconfigured. Fix: pass `async: false` to bypass the message bus and write directly to `audit_log` table.

- [x] 1.1 **Update `processApproval()` audit log call to sync**
  - File: `bidding-api/src/Service/FlexSwitchApprovalService.php`
  - Find the `$this->auditLogService->log(` call in `processApproval()` (around line 296)
  - Add `async: false` parameter to the call
  - Keep all other parameters (action, entityType, entityId, oldData, newData, description) unchanged

- [x] 1.2 **Update `submitRequest()` audit log call to sync**
  - File: `bidding-api/src/Service/FlexSwitch/FlexSwitchService.php`
  - Find the `$this->auditLogService->log(` call in `submitRequest()` (around line 817)
  - Add `async: false` parameter to the call

- [x] 1.3 **Update `cancelRequest()` audit log call to sync**
  - File: `bidding-api/src/Service/FlexSwitch/FlexSwitchService.php`
  - Find the `$this->auditLogService->log(` call in `cancelRequest()` (around line 664)
  - Add `async: false` parameter to the call

## 2. Diagnose Silent Errors (if entries still missing after task 1)

- [ ] 2.1 **Check application logs for audit log errors**
  - Search Symfony log files (`var/log/dev.log` or `var/log/prod.log`) for:
    - `"Failed to save audit log"` — indicates `AuditLogService::log()` caught an exception
    - `"Failed to save audit log from message"` — indicates `AuditLogMessageHandler` caught an exception
  - If errors found, the stack trace will reveal the root cause (serialization, DB constraint, etc.)

- [ ] 2.2 **Query `audit_log` table directly**
  - Run: `SELECT id, action, entity_type, entity_id, description, created_at FROM audit_log WHERE entity_type = 'FlexSwitchRequest' ORDER BY created_at DESC LIMIT 10;`
  - If rows exist → the issue is frontend display, not persistence
  - If no rows → the `log()` call is failing silently or not being reached

- [ ] 2.3 **Verify Symfony Messenger transport status (informational)**
  - Check `MESSENGER_TRANSPORT_DSN` env var value
  - If `doctrine://default`: check `messenger_messages` table for queued `AuditLogMessage` entries
  - If `sync://`: messages are processed synchronously (task 1 fix would be redundant but harmless)

## 3. Verification — Confirm Audit Logs Appear on UI

- [ ] 3.1 **Test student submit → audit log**
  - Login as student, submit a flex switch request via `POST /v2/api/student/flex-switch/request`
  - Query DB: `SELECT * FROM audit_log WHERE action = 'CREATE FlexSwitchRequest' ORDER BY created_at DESC LIMIT 1;`
  - Confirm row exists with correct `entity_type`, `entity_id`, `old_data`, `new_data`, `description`
  - Go to Monitoring & Analytics > Audit Logs page and confirm the entry is visible

- [ ] 3.2 **Test PM approve/reject → audit log**
  - Login as PM, approve or reject a pending flex switch request via `POST /v2/api/dashboard/flex-switch/approval-requests/{id}/process`
  - Query DB: `SELECT * FROM audit_log WHERE action LIKE '%FlexSwitchRequest' AND action != 'CREATE FlexSwitchRequest' ORDER BY created_at DESC LIMIT 1;`
  - Confirm row exists with correct status transition in `old_data` / `new_data`
  - Go to Audit Logs page and confirm the entry is visible

- [ ] 3.3 **Test student cancel → audit log**
  - Login as student, cancel a pending flex switch request
  - Query DB: `SELECT * FROM audit_log WHERE action = 'CANCEL FlexSwitchRequest' ORDER BY created_at DESC LIMIT 1;`
  - Confirm row exists
  - Go to Audit Logs page and confirm the entry is visible

## 4. Endpoint Clarification (no code change needed)

- [x] 4.1 **Document: GET `/v2/api/dashboard/flex-switch/approval-requests` does NOT create audit logs**
  - This is a read-only list endpoint (`FlexSwitchApprovalController::getApprovalRequests()`)
  - Audit logs are only created for mutations: submit (POST), approve/reject (POST), cancel (POST)
  - The approval action endpoint is `POST /v2/api/dashboard/flex-switch/approval-requests/{id}/process`
