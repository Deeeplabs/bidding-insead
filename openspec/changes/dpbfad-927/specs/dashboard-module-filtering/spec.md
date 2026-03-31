## ADDED Requirements

### Requirement: Backend Dashboard Module Filtering
The `/v2/api/student/flex-switch/calendar` backend API endpoint serving the student dashboard calendar MUST implicitly filter the returned modules to ensure only those related to the student's programme and allowed campuses are included in the payload. The allowed campus set includes the student's home campus, any exchange campuses, and the FLEX campus.

#### Scenario: Displaying modules for a student's home campus
- **WHEN** the frontend requests the dashboard calendar module list via `/v2/api/student/flex-switch/calendar`
- **THEN** the backend evaluates the student's programme and home campus and returns ONLY modules matching the student's allowed campus set, without requiring any special request parameters from the client.

#### Scenario: Including FLEX campus modules for all students
- **WHEN** the frontend requests the dashboard calendar module list via `/v2/api/student/flex-switch/calendar`
- **THEN** the backend MUST always include modules from the FLEX campus (short_name = "FLEX") in addition to the student's home campus and exchange campuses, so flex/virtual delivery modules are visible to all students regardless of their physical location.

#### Scenario: Including exchange campus modules
- **WHEN** a student has active Exchange records with assigned P3, P4, or P5 campuses
- **THEN** the backend MUST include modules from those exchange campuses in addition to the home campus and FLEX campus.

#### Scenario: Graceful handling when FLEX campus does not exist
- **WHEN** no campus with short_name "FLEX" exists in the database
- **THEN** the backend MUST proceed without error, filtering only by home campus and exchange campuses as before.

#### Scenario: Independence from Switch Request
- **WHEN** the student interacts with the Switch Request interface
- **THEN** they can still access other modules, as the switch request relies on independent data-fetching logic that is unaffected by this dashboard filter.
