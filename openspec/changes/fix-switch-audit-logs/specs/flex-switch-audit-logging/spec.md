## ADDED Requirements

### Requirement: Audit log for FlexSwitch approval processing

When a Program Manager approves or rejects a flex switch request via `POST /dashboard/flex-switch/approval-requests/{id}/process`, an audit log entry must be **synchronously persisted** to the `audit_log` table, recording the action, the old and new status, the approver, and the affected student/course details.

#### Scenario: PM approves a flex switch request
- **GIVEN** a flex switch request with ID 42 is in `pending` status for student "John Doe" switching from class 101 to class 202
- **WHEN** PM user approves the request with remarks "Schedule conflict verified"
- **THEN** an audit log entry is **immediately written** to the `audit_log` table (synchronous, `async: false`) with:
  - `action`: `APPROVE FlexSwitchRequest`
  - `entityType`: `FlexSwitchRequest`
  - `entityId`: 42
  - `oldData`: `{ "status": "pending", "student_id": "0652934", "from_class_id": 101, "to_class_id": 202, "from_course_id": "...", "to_course_id": "..." }`
  - `newData`: `{ "status": "approved", "approved_by": <approver_id>, "approved_at": <timestamp>, "remarks": "Schedule conflict verified" }`
  - `description`: a human-readable summary including student name and action
- **AND** the entry is visible on the Monitoring & Analytics > Audit Logs page without delay

#### Scenario: PM rejects a flex switch request
- **GIVEN** a flex switch request with ID 43 is in `pending` status
- **WHEN** PM user rejects the request with remarks "Insufficient justification"
- **THEN** an audit log entry is **immediately written** to the `audit_log` table (synchronous, `async: false`) with:
  - `action`: `REJECT FlexSwitchRequest`
  - `entityType`: `FlexSwitchRequest`
  - `entityId`: 43
  - `oldData`: `{ "status": "pending", "student_id": "...", "from_class_id": ..., "to_class_id": ..., "from_course_id": "...", "to_course_id": "..." }`
  - `newData`: `{ "status": "rejected", "approved_by": <approver_id>, "approved_at": <timestamp>, "remarks": "Insufficient justification" }`

### Requirement: Audit log for student flex switch request submission

When a student submits a new flex switch request via `POST /student/flex-switch/request`, an audit log entry must be **synchronously persisted** to the `audit_log` table.

#### Scenario: Student submits a flex switch request
- **GIVEN** student "Jane Smith" (ID: 0652935) wants to switch from class 101 to class 202
- **WHEN** the student submits the request with reason "Scheduling conflict with elective"
- **THEN** an audit log entry is **immediately written** to the `audit_log` table (synchronous, `async: false`) with:
  - `action`: `CREATE FlexSwitchRequest`
  - `entityType`: `FlexSwitchRequest`
  - `entityId`: <new_request_id>
  - `oldData`: null
  - `newData`: `{ "student_id": "0652935", "from_class_id": 101, "to_class_id": 202, "from_course_id": "...", "to_course_id": "...", "reason": "Scheduling conflict with elective", "status": "pending" }`
  - `description`: a human-readable summary of the submission
- **AND** the entry is visible on the Audit Logs page without delay

### Requirement: Audit log for student flex switch request cancellation

When a student cancels a pending flex switch request via `POST /student/flex-switch/request/{id}/cancel`, an audit log entry must be **synchronously persisted** to the `audit_log` table.

#### Scenario: Student cancels a pending flex switch request
- **GIVEN** student "Jane Smith" has a pending flex switch request with ID 44
- **WHEN** the student cancels the request with reason "No longer needed"
- **THEN** an audit log entry is **immediately written** to the `audit_log` table (synchronous, `async: false`) with:
  - `action`: `CANCEL FlexSwitchRequest`
  - `entityType`: `FlexSwitchRequest`
  - `entityId`: 44
  - `oldData`: `{ "status": "pending", "student_id": "...", "from_class_id": ..., "to_class_id": ... }`
  - `newData`: `{ "status": "cancelled", "cancelled_by": "0652935", "cancelled_at": <timestamp>, "cancellation_reason": "No longer needed" }`

### Requirement: Audit log for PM course adjustment

When a PM saves course adjustment changes via `POST /flex-switch/course-adjustment/{course_id}/{class_id}`, audit log entries must be **synchronously persisted** for each changed entity.

#### Scenario: PM updates course details via course adjustment
- **GIVEN** a course "Finance 101" (ID: 007304) with class ID 19142
- **WHEN** PM user updates the course name, credit, or information link
- **THEN** an audit log entry is **immediately written** (synchronous, `async: false`) with:
  - `action`: `UPDATE Course`
  - `entityType`: `Course`
  - `entityId`: course ID
  - `oldData`/`newData`: course fields before/after change

#### Scenario: PM updates class details via course adjustment
- **GIVEN** the same class
- **WHEN** PM user updates deadline, campus, delivery mode, instructor, assistant email, or conflicts
- **THEN** an audit log entry is **immediately written** with:
  - `action`: `UPDATE Class`
  - `entityType`: `Class`

#### Scenario: PM updates seat capacity via course adjustment
- **WHEN** PM user changes the seat capacity
- **THEN** an audit log entry is **immediately written** with:
  - `action`: `UPDATE ClassPromotion`
  - `entityType`: `ClassPromotion`

#### Scenario: PM creates or updates flex course adjustment
- **WHEN** PM user changes switch seats or fallbacks
- **THEN** an audit log entry is **immediately written** with:
  - `action`: `CREATE FlexCourseAdjustment` or `UPDATE FlexCourseAdjustment`
  - `entityType`: `FlexCourseAdjustment`

### Requirement: Audit log for PM FlexSwitch configuration

When a PM creates or updates a FlexSwitch configuration, audit log entries must be **synchronously persisted**.

#### Scenario: PM creates a new FlexSwitch configuration
- **WHEN** PM creates a configuration for a promotion
- **THEN** an audit log entry is **immediately written** with:
  - `action`: `CREATE FlexSwitchConfiguration`
  - `entityType`: `FlexSwitchConfiguration`

#### Scenario: PM updates an existing FlexSwitch configuration
- **WHEN** PM updates a configuration for a promotion
- **THEN** an audit log entry is **immediately written** with:
  - `action`: `UPDATE FlexSwitchConfiguration`
  - `entityType`: `FlexSwitchConfiguration`

### Requirement: Audit log entries appear in existing Audit Logs view

No frontend changes needed — audit log entries for FlexSwitch operations appear in the existing Monitoring & Analytics > Audit Logs page automatically because they use the same `audit_log` table and `AuditLogService`.

#### Scenario: Viewing FlexSwitch audit logs in Audit Logs page
- **GIVEN** a PM has approved requests and saved course adjustments
- **WHEN** another admin views the Audit Logs page
- **THEN** all FlexSwitch entries appear in the log list with correct action, entity type, description, and timestamp

#### Scenario: Searching for FlexSwitch audit logs
- **GIVEN** FlexSwitch audit log entries exist in the `audit_log` table
- **WHEN** an admin searches for "FlexSwitch" in the Audit Logs search box
- **THEN** entries with actions like `APPROVE FlexSwitchRequest`, `CREATE FlexSwitchRequest`, `UPDATE Course`, `UPDATE Class`, etc. are returned

### Requirement: GET endpoints do NOT create audit logs

Read-only endpoints do NOT create audit log entries. Only mutation endpoints (POST) create audit logs.

#### Scenario: Viewing the approval requests list
- **WHEN** a PM calls `GET /v2/api/dashboard/flex-switch/approval-requests?page=1&limit=5`
- **THEN** no audit log entry is created

#### Scenario: Viewing course adjustment details
- **WHEN** a PM calls `GET /v2/api/flex-switch/course-adjustment/{course_id}/{class_id}`
- **THEN** no audit log entry is created

### Requirement: No impact on existing behavior

#### Scenario: Existing FlexSwitch operations continue to work normally
- **WHEN** the audit logging uses synchronous mode (`async: false`)
- **THEN** all existing FlexSwitch operations (submit, approve, reject, cancel, course adjustment, configuration) continue to function identically
- **AND** API response shapes remain unchanged
- **AND** no new database migrations are required
- **AND** the only additional latency is a single synchronous DB write per audit log entry (~1-5ms)
