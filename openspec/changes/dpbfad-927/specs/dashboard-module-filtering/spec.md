## ADDED Requirements

### Requirement: Backend Dashboard Module Filtering
The `/v2/api/student/flex-switch/calendar` backend API endpoint serving the student dashboard calendar MUST implicitly filter the returned modules to ensure only those related to the student's programme and home campus are included in the payload.

#### Scenario: Displaying modules for a specific student's context
- **WHEN** the frontend requests the dashboard calendar module list via `/v2/api/student/flex-switch/calendar`
- **THEN** the backend evaluates the student's programme and home campus (including active exchanges) and returns ONLY matching modules, without requiring any special request parameters from the client.

#### Scenario: Independence from Switch Request
- **WHEN** the student interacts with the Switch Request interface
- **THEN** they can still access other modules, as the switch request relies on independent data-fetching logic that is unaffected by this dashboard filter.
