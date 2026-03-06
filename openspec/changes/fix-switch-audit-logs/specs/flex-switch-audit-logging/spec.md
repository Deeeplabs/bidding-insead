## ADDED Requirements

### Requirement: Audit log for FlexSwitch approval processing

When a Program Manager approves or rejects a flex switch request via `POST /dashboard/flex-switch/approval-requests/{id}/process`, an audit log entry must be created recording the action, the old and new status, the approver, and the affected student/course details.

#### Scenario: PM approves a flex switch request
- **GIVEN** a flex switch request with ID 42 is in `pending` status for student "John Doe" switching from class 101 to class 202
- **WHEN** PM user approves the request with remarks "Schedule conflict verified"
- **THEN** an audit log entry is created with:
  - `action`: `APPROVE FlexSwitchRequest`
  - `entityType`: `FlexSwitchRequest`
  - `entityId`: 42
  - `oldData`: `{ "status": "pending", "student_id": "0652934", "from_class_id": 101, "to_class_id": 202 }`
  - `newData`: `{ "status": "approved", "approved_by": <approver_id>, "approved_at": <timestamp>, "remarks": "Schedule conflict verified" }`
  - `description`: a human-readable summary including student name and action

#### Scenario: PM rejects a flex switch request
- **GIVEN** a flex switch request with ID 43 is in `pending` status
- **WHEN** PM user rejects the request with remarks "Insufficient justification"
- **THEN** an audit log entry is created with:
  - `action`: `REJECT FlexSwitchRequest`
  - `entityType`: `FlexSwitchRequest`
  - `entityId`: 43
  - `oldData`: `{ "status": "pending", "student_id": "...", "from_class_id": ..., "to_class_id": ... }`
  - `newData`: `{ "status": "rejected", "approved_by": <approver_id>, "approved_at": <timestamp>, "remarks": "Insufficient justification" }`

### Requirement: Audit log for student flex switch request submission

When a student submits a new flex switch request via `POST /student/flex-switch/request`, an audit log entry must be created.

#### Scenario: Student submits a flex switch request
- **GIVEN** student "Jane Smith" (ID: 0652935) wants to switch from class 101 to class 202
- **WHEN** the student submits the request with reason "Scheduling conflict with elective"
- **THEN** an audit log entry is created with:
  - `action`: `CREATE FlexSwitchRequest`
  - `entityType`: `FlexSwitchRequest`
  - `entityId`: <new_request_id>
  - `oldData`: null
  - `newData`: `{ "student_id": "0652935", "from_class_id": 101, "to_class_id": 202, "from_course_id": "...", "to_course_id": "...", "reason": "Scheduling conflict with elective", "status": "pending" }`
  - `description`: a human-readable summary of the submission

### Requirement: Audit log for student flex switch request cancellation

When a student cancels a pending flex switch request via `POST /student/flex-switch/request/{id}/cancel`, an audit log entry must be created.

#### Scenario: Student cancels a pending flex switch request
- **GIVEN** student "Jane Smith" has a pending flex switch request with ID 44
- **WHEN** the student cancels the request with reason "No longer needed"
- **THEN** an audit log entry is created with:
  - `action`: `CANCEL FlexSwitchRequest`
  - `entityType`: `FlexSwitchRequest`
  - `entityId`: 44
  - `oldData`: `{ "status": "pending" }`
  - `newData`: `{ "status": "cancelled", "cancelled_by": "0652935", "cancelled_at": <timestamp>, "cancellation_reason": "No longer needed" }`

### Requirement: Audit log entries appear in existing Audit Logs view

No frontend changes needed — audit log entries for FlexSwitch operations should appear in the existing Monitoring & Analytics > Audit Logs page automatically because they use the same `audit_log` table and `AuditLogService`.

#### Scenario: Viewing FlexSwitch audit logs in Audit Logs page
- **GIVEN** a PM has approved several flex switch requests
- **WHEN** another admin views the Audit Logs page
- **THEN** the FlexSwitch approval/rejection entries appear in the log list with correct action, entity type, description, and timestamp

### Requirement: No impact on existing behavior

#### Scenario: Existing FlexSwitch operations continue to work normally
- **WHEN** the audit logging is added to FlexSwitch services
- **THEN** all existing FlexSwitch operations (submit, approve, reject, cancel) continue to function identically
- **AND** API response shapes remain unchanged
- **AND** no new database migrations are required
