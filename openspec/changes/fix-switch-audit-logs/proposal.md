## Why

When a Program Manager (PM) interacts with the Flex Switch module — specifically approving/rejecting student switch requests — none of these interactions are recorded in the system Audit Logs. Similarly, when students submit or cancel flex switch requests, these actions are also missing from audit trails.

This is a compliance and traceability gap. The audit log system exists and is actively used for other modules (user management, campaign phases, simulations, course adjustments, etc.), but the FlexSwitch approval workflow was never wired to it.

The root cause: `FlexSwitchApprovalService.processApproval()` does not call `AuditLogService` at all. The student-side `FlexSwitchService.submitRequest()` and `FlexSwitchService.cancelRequest()` also lack audit logging. While the PM-side `PM\FlexSwitchService.createConfiguration()` and `saveCourseAdjustment()` DO have audit logging, the approval/reject workflow and student request lifecycle are completely unaudited.

## What Changes

Add `AuditLogService` calls to all FlexSwitch mutation operations that currently lack audit logging:

1. **FlexSwitchApprovalService.processApproval()** — Log when a PM approves or rejects a flex switch request, including old status (pending), new status (approved/rejected), approver details, remarks, student and course info.

2. **FlexSwitchService.submitRequest()** (student side) — Log when a student submits a new flex switch request, including from/to class details and reason.

3. **FlexSwitchService.cancelRequest()** (student side) — Log when a student cancels a pending flex switch request, including cancellation reason.

## Capabilities

### New Capabilities
- `flex-switch-audit-logging`: Add audit log entries for FlexSwitch approval processing (approve/reject) and student request lifecycle (submit/cancel)

### Modified Capabilities
<!-- No existing spec-level behavior changes — this adds audit logging to existing operations -->

## Impact

- **API (bidding-api)**:
  - `src/Service/FlexSwitchApprovalService.php` — Add `AuditLogService` dependency and audit log calls in `processApproval()`
  - `src/Service/FlexSwitch/FlexSwitchService.php` — Add `AuditLogService` dependency and audit log calls in `submitRequest()` and `cancelRequest()`
- **No API response changes** — Audit logging is a side-effect, no response shape changes
- **No migration required** — Uses existing `audit_log` table
- **No breaking changes** — Purely additive behavior
