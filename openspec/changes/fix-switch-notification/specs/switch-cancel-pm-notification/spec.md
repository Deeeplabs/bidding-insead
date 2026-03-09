## ADDED Requirements

### Requirement: PM notification on student cancel

When a student cancels their pending flex switch request, all Program Managers associated with the student's program must receive an in-app notification informing them of the cancellation.

#### Scenario: Student cancels a pending switch request with PMs configured
- **GIVEN** a student has a pending flex switch request
- **AND** the student belongs to a promotion with a program
- **AND** the program has one or more Program Managers with associated User accounts
- **WHEN** the student calls `PUT /student/flex-switch/my-requests/{id}/cancel` with a valid cancellation reason
- **THEN** the request is cancelled successfully (status → `cancelled`)
- **AND** a `CUSTOM_ANNOUNCEMENT` notification is sent to each PM user with:
  - Title: `"Flex Switch Request Cancelled"`
  - Body: `"Student {firstName} {lastName} has cancelled their switch request from {fromCourseName} to {toCourseName}. Reason: {cancellationReason}."`
  - Data includes: `request_id`, `student_id`, `from_course`, `to_course`, `cancellation_reason`

#### Scenario: Student cancels but no PMs are configured
- **GIVEN** a student has a pending flex switch request
- **AND** the student's program has no Program Managers (or PMs have no associated User)
- **WHEN** the student cancels the request
- **THEN** the request is still cancelled successfully
- **AND** no notification error occurs (graceful handling)

#### Scenario: Student cancels but has no promotion/program
- **GIVEN** a student has a pending flex switch request
- **AND** the student has no promotion or the promotion has no program
- **WHEN** the student cancels the request
- **THEN** the request is still cancelled successfully
- **AND** no notification is sent (no PM recipients to notify)

#### Scenario: Non-pending request cannot be cancelled
- **GIVEN** a request has status other than `pending` (e.g., `approved`, `rejected`, `cancelled`)
- **WHEN** the student attempts to cancel it
- **THEN** a 400 error is returned with code `INVALID_STATUS`
- **AND** no notification is sent

### Requirement: Notification data fields

The notification sent to PMs must include sufficient data for context:

#### Scenario: Notification payload structure
- **WHEN** a PM receives a cancel notification
- **THEN** the notification data includes:
  - `announcement_title`: `"Flex Switch Request Cancelled"`
  - `announcement_body`: Human-readable summary with student name, courses, and reason
  - `request_id`: The flex switch request ID
  - `student_id`: The cancelling student's ID
  - `from_course`: Source course name
  - `to_course`: Target course name
  - `cancellation_reason`: The reason provided by the student
