# Fix: Flex Switch Audit Logs Not Appearing on Audit Logs Page

## Problem

Audit log entries for FlexSwitch operations were **not visible** on the Monitoring & Analytics > Audit Logs page, despite `AuditLogService->log()` calls being present in the code.

Reported endpoints affected:
- `POST /v2/api/student/flex-switch/request` — student submits a flex switch request
- `POST /v2/api/dashboard/flex-switch/approval-requests/{id}/process` — PM approves/rejects a request
- `POST /v2/api/flex-switch/course-adjustment/{course_id}/{class_id}` — PM saves course adjustment changes

> **Note:** GET endpoints (`/approval-requests?page=1&limit=5`, `/course-adjustment/{id}/{id}`) are read-only — they do NOT create audit logs by design.

### Root Cause

All 10 `AuditLogService->log()` calls across 3 FlexSwitch services used the default `async: true`, which dispatches `AuditLogMessage` to the Symfony Messenger `async` transport (`config/packages/messenger.yaml`). The transport DSN depends on the `MESSENGER_TRANSPORT_DSN` env var — if this is a queue-based transport (Doctrine, Redis, AMQP), a running consumer worker (`php bin/console messenger:consume async`) is required. If the worker is not running or the transport is misconfigured, messages pile up in the queue but never persist to the `audit_log` table.

Additionally, both `AuditLogService::log()` and `AuditLogMessageHandler::__invoke()` catch ALL exceptions silently (logging to file only), making errors invisible to users and API responses.

## Solution

Switched all 10 FlexSwitch audit log calls to **synchronous mode** (`async: false`), bypassing the Messenger bus entirely. This writes audit entries directly to the `audit_log` table in the same request via `AuditLogRepository::save()`.

### Changes Made

**Modified File:**

1. **`src/Service/FlexSwitchApprovalService.php`** (1 call)
   - `processApproval()` — added `async: false`
   - Logs `APPROVE FlexSwitchRequest` or `REJECT FlexSwitchRequest` synchronously

2. **`src/Service/FlexSwitch/FlexSwitchService.php`** (2 calls)
   - `submitRequest()` — added `async: false`, logs `CREATE FlexSwitchRequest` synchronously
   - `cancelRequest()` — added `async: false`, logs `CANCEL FlexSwitchRequest` synchronously

3. **`src/Service/FlexSwitch/PM/FlexSwitchService.php`** (7 calls)
   - `createConfiguration()` — added `async: false` to both `CREATE FlexSwitchConfiguration` and `UPDATE FlexSwitchConfiguration` calls
   - `saveCourseAdjustment()` — added `async: false` to all 5 audit log calls:
     - `UPDATE Course` — when course name/credit/info link changes
     - `UPDATE Class` — when deadline/campus/mode/instructor/assistant/conflicts change
     - `UPDATE ClassPromotion` — when seat capacity changes
     - `CREATE FlexCourseAdjustment` — when new adjustment is created
     - `UPDATE FlexCourseAdjustment` — when existing adjustment switch seats/fallbacks change

## Audit Log Actions Summary

| Operation                    | Action                           | Entity Type              | Service Method                              |
|------------------------------|----------------------------------|--------------------------|---------------------------------------------|
| PM approves request          | `APPROVE FlexSwitchRequest`      | `FlexSwitchRequest`      | `FlexSwitchApprovalService.processApproval`  |
| PM rejects request           | `REJECT FlexSwitchRequest`       | `FlexSwitchRequest`      | `FlexSwitchApprovalService.processApproval`  |
| Student submits request      | `CREATE FlexSwitchRequest`       | `FlexSwitchRequest`      | `FlexSwitchService.submitRequest`            |
| Student cancels request      | `CANCEL FlexSwitchRequest`       | `FlexSwitchRequest`      | `FlexSwitchService.cancelRequest`            |
| PM creates config            | `CREATE FlexSwitchConfiguration` | `FlexSwitchConfiguration`| `PM\FlexSwitchService.createConfiguration`   |
| PM updates config            | `UPDATE FlexSwitchConfiguration` | `FlexSwitchConfiguration`| `PM\FlexSwitchService.createConfiguration`   |
| PM updates course            | `UPDATE Course`                  | `Course`                 | `PM\FlexSwitchService.saveCourseAdjustment`  |
| PM updates class             | `UPDATE Class`                   | `Class`                  | `PM\FlexSwitchService.saveCourseAdjustment`  |
| PM updates seat capacity     | `UPDATE ClassPromotion`          | `ClassPromotion`         | `PM\FlexSwitchService.saveCourseAdjustment`  |
| PM creates adjustment        | `CREATE FlexCourseAdjustment`    | `FlexCourseAdjustment`   | `PM\FlexSwitchService.saveCourseAdjustment`  |
| PM updates adjustment        | `UPDATE FlexCourseAdjustment`    | `FlexCourseAdjustment`   | `PM\FlexSwitchService.saveCourseAdjustment`  |

## Impact

- **Audit Compliance:** All FlexSwitch state-changing operations now reliably persist to the `audit_log` table
- **No API Changes:** Purely backend change — no response shape modifications
- **No Migrations:** Uses existing `audit_log` table infrastructure
- **No Breaking Changes:** All existing FlexSwitch operations continue to function identically
- **Synchronous Logging:** Adds a small DB write per audit entry (~1-5ms). Acceptable for low-frequency FlexSwitch operations

## Verification

- [ ] Query DB: `SELECT * FROM audit_log WHERE entity_type IN ('FlexSwitchRequest', 'Course', 'Class', 'ClassPromotion', 'FlexCourseAdjustment', 'FlexSwitchConfiguration') ORDER BY created_at DESC LIMIT 10;`
- [ ] PM approval/rejection creates audit log entries
- [ ] Student submit creates audit log entry
- [ ] Student cancel creates audit log entry
- [ ] PM course adjustment creates audit log entries (up to 5 per save)
- [ ] PM configuration create/update creates audit log entry
- [ ] All entries visible in Monitoring & Analytics > Audit Logs page
