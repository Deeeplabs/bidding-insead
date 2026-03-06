# Fix: Add Audit Logging to Flex Switch Operations

## Problem

When a Programme Manager (PM) interacts with the Flex Switch module — specifically approving or rejecting student switch requests — none of these actions were recorded in the system Audit Logs. Similarly, student-side operations (submitting and cancelling flex switch requests) were also completely unaudited.

This was a compliance and traceability gap. While the `AuditLogService` was already used extensively across other modules (user management, campaigns, simulations), and even partially within Flex Switch itself (`createConfiguration` and `saveCourseAdjustment` were audited), the **approval workflow** and **student request lifecycle** were never wired to it.

### Root Cause

- `FlexSwitchApprovalService.processApproval()` — did not inject or call `AuditLogService`
- `FlexSwitchService.submitRequest()` (student side) — did not inject or call `AuditLogService`
- `FlexSwitchService.cancelRequest()` (student side) — did not inject or call `AuditLogService`

## Solution

Added `AuditLogService` dependency injection and audit log calls to all FlexSwitch mutation operations that previously lacked audit logging. Followed the exact same pattern already established in `PM\FlexSwitchService.createConfiguration()` and `saveCourseAdjustment()`.

### Changes Made

**Modified File:**

1. **`src/Service/FlexSwitchApprovalService.php`**
   - Added `AuditLogService` import and constructor dependency injection
   - Added audit log call in `processApproval()` after successful flush:
     - Logs `APPROVE FlexSwitchRequest` or `REJECT FlexSwitchRequest` action
     - Records old data (pending status, student/course IDs) and new data (approved/rejected status, approver, remarks)
     - Includes human-readable description with PM name, student name, and request ID

2. **`src/Service/FlexSwitch/FlexSwitchService.php`**
   - Added `AuditLogService` import and constructor dependency injection
   - Added audit log call in `submitRequest()` after successful save:
     - Logs `CREATE FlexSwitchRequest` action
     - Records new data (student ID, from/to class and course IDs, reason, pending status)
     - Includes description with student name and course names
   - Added audit log call in `cancelRequest()` after successful save:
     - Logs `CANCEL FlexSwitchRequest` action
     - Records old data (pending status) and new data (cancelled status, cancellation reason, timestamp)
     - Includes description with student name and request ID

## Audit Log Actions Summary

| Operation                 | Action                         | Entity Type          | Data Captured                                                  |
|---------------------------|--------------------------------|----------------------|----------------------------------------------------------------|
| PM approves request       | `APPROVE FlexSwitchRequest`    | `FlexSwitchRequest`  | Status transition, approver, student/course info, remarks      |
| PM rejects request        | `REJECT FlexSwitchRequest`     | `FlexSwitchRequest`  | Status transition, approver, student/course info, remarks      |
| Student submits request   | `CREATE FlexSwitchRequest`     | `FlexSwitchRequest`  | From/to class/course IDs, reason, student info                 |
| Student cancels request   | `CANCEL FlexSwitchRequest`     | `FlexSwitchRequest`  | Status transition, cancellation reason, student info           |

## Impact

- **Audit Compliance:** All FlexSwitch state-changing operations are now fully recorded in the `audit_log` table
- **No API Changes:** Purely additive backend change — no response shape modifications
- **No Migrations:** Uses existing `audit_log` table infrastructure
- **No Breaking Changes:** All existing FlexSwitch operations continue to function identically
- **Async Logging:** Uses the default async message bus for audit log dispatch (no request latency impact)

## Testing

- [x] PM approval/rejection creates audit log entries with correct action, entity type, old/new data
- [x] Student submit creates audit log entry with course details
- [x] Student cancel creates audit log entry with status transition
- [x] Existing audit log entries (configuration, course adjustment) remain unaffected
- [x] All new entries visible in Monitoring & Analytics > Audit Logs page
