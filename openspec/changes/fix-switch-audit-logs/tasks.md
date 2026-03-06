## 1. Switch Audit Log Calls to Synchronous Mode (Root Cause Fix)

The existing audit log calls use the default `async: true`, which dispatches `AuditLogMessage` to the Symfony Messenger `async` transport. Messages are queued but may not be consumed if the worker is down or the transport is misconfigured. Fix: pass `async: false` to bypass the message bus and write directly to `audit_log` table.

### 1A. FlexSwitchApprovalService (1 call)

- [x] 1.1 **Update `processApproval()` audit log call to sync**
  - File: `bidding-api/src/Service/FlexSwitchApprovalService.php`
  - Add `async: false` parameter to the `$this->auditLogService->log(` call in `processApproval()`

### 1B. FlexSwitchService — student side (2 calls)

- [x] 1.2 **Update `submitRequest()` audit log call to sync**
  - File: `bidding-api/src/Service/FlexSwitch/FlexSwitchService.php`
  - Add `async: false` parameter to the `$this->auditLogService->log(` call in `submitRequest()`

- [x] 1.3 **Update `cancelRequest()` audit log call to sync**
  - File: `bidding-api/src/Service/FlexSwitch/FlexSwitchService.php`
  - Add `async: false` parameter to the `$this->auditLogService->log(` call in `cancelRequest()`

### 1C. PM\FlexSwitchService — createConfiguration (2 calls)

- [x] 1.4 **Update `createConfiguration()` UPDATE audit log call to sync**
  - File: `bidding-api/src/Service/FlexSwitch/PM/FlexSwitchService.php`
  - Find the `$this->auditLogService->log(` call for `UPDATE FlexSwitchConfiguration` (around line 771)
  - Add `async: false` parameter

- [x] 1.5 **Update `createConfiguration()` CREATE audit log call to sync**
  - File: `bidding-api/src/Service/FlexSwitch/PM/FlexSwitchService.php`
  - Find the `$this->auditLogService->log(` call for `CREATE FlexSwitchConfiguration` (around line 780)
  - Add `async: false` parameter

### 1D. PM\FlexSwitchService — saveCourseAdjustment (5 calls)

- [x] 1.6 **Update `saveCourseAdjustment()` UPDATE Course audit log call to sync**
  - File: `bidding-api/src/Service/FlexSwitch/PM/FlexSwitchService.php`
  - Find the `$this->auditLogService->log(` call for `UPDATE Course` (around line 1218)
  - Add `async: false` parameter

- [x] 1.7 **Update `saveCourseAdjustment()` UPDATE Class audit log call to sync**
  - File: `bidding-api/src/Service/FlexSwitch/PM/FlexSwitchService.php`
  - Find the `$this->auditLogService->log(` call for `UPDATE Class` (around line 1232)
  - Add `async: false` parameter

- [x] 1.8 **Update `saveCourseAdjustment()` UPDATE ClassPromotion audit log call to sync**
  - File: `bidding-api/src/Service/FlexSwitch/PM/FlexSwitchService.php`
  - Find the `$this->auditLogService->log(` call for `UPDATE ClassPromotion` (around line 1244)
  - Add `async: false` parameter

- [x] 1.9 **Update `saveCourseAdjustment()` CREATE FlexCourseAdjustment audit log call to sync**
  - File: `bidding-api/src/Service/FlexSwitch/PM/FlexSwitchService.php`
  - Find the `$this->auditLogService->log(` call for `CREATE FlexCourseAdjustment` (around line 1262)
  - Add `async: false` parameter

- [x] 1.10 **Update `saveCourseAdjustment()` UPDATE FlexCourseAdjustment audit log call to sync**
  - File: `bidding-api/src/Service/FlexSwitch/PM/FlexSwitchService.php`
  - Find the `$this->auditLogService->log(` call for `UPDATE FlexCourseAdjustment` (around line 1273)
  - Add `async: false` parameter

## 2. Diagnose Silent Errors (if entries still missing after task 1)

- [ ] 2.1 **Check application logs for audit log errors**
  - Search Symfony log files (`var/log/dev.log` or `var/log/prod.log`) for:
    - `"Failed to save audit log"` — indicates `AuditLogService::log()` caught an exception
    - `"Failed to save audit log from message"` — indicates `AuditLogMessageHandler` caught an exception

- [ ] 2.2 **Query `audit_log` table directly**
  - Run: `SELECT id, action, entity_type, entity_id, description, created_at FROM audit_log WHERE entity_type IN ('FlexSwitchRequest', 'Course', 'Class', 'ClassPromotion', 'FlexCourseAdjustment', 'FlexSwitchConfiguration') ORDER BY created_at DESC LIMIT 10;`
  - If rows exist → the issue is frontend display, not persistence
  - If no rows → the `log()` call is failing silently or not being reached

## 3. Verification — Confirm Audit Logs Appear on UI

- [ ] 3.1 **Test student submit → audit log**
  - Login as student, submit a flex switch request via `POST /v2/api/student/flex-switch/request`
  - Confirm entry visible on Audit Logs page

- [ ] 3.2 **Test PM approve/reject → audit log**
  - Login as PM, approve or reject via `POST /v2/api/dashboard/flex-switch/approval-requests/{id}/process`
  - Confirm entry visible on Audit Logs page

- [ ] 3.3 **Test student cancel → audit log**
  - Login as student, cancel a pending flex switch request
  - Confirm entry visible on Audit Logs page

- [ ] 3.4 **Test PM course adjustment → audit log**
  - Login as PM, save course adjustment via `POST /v2/api/flex-switch/course-adjustment/{course_id}/{class_id}`
  - Confirm entries visible on Audit Logs page (up to 5 entries per save depending on fields changed)

## 4. Endpoint Clarification (no code change needed)

- [x] 4.1 **Document: GET endpoints do NOT create audit logs**
  - `GET /v2/api/dashboard/flex-switch/approval-requests` — read-only list
  - `GET /v2/api/flex-switch/course-adjustment/{course_id}/{class_id}` — read-only get
  - Audit logs are only created for POST mutation endpoints
