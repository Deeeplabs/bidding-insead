## ADDED Requirements

### Requirement: Add/Drop submission SHALL tolerate missing StudentData
The system SHALL allow Add/Drop submission processing to continue when `Student::getStudentData()` is null. Financial values used in submission summary calculations MUST default to safe numeric values (`remaining_capital = 0`, `credits_taken = 0.0`) instead of causing runtime errors.

#### Scenario: Build Add/Drop response when student financial relation is missing
- **GIVEN** an active campaign Add/Drop submission where the student has no related `StudentData`
- **WHEN** the backend builds the Add/Drop submission summary
- **THEN** the request SHALL complete without null-dereference exceptions
- **AND** `bid_points_remaining` SHALL be computed using `0` as base remaining capital
- **AND** `total_credits_after` SHALL be computed using `0.0` as base credits taken.

### Requirement: Add/Drop audit logging SHALL capture fallback financial snapshot
The system SHALL persist Add/Drop audit log entries even when `StudentData` is null, using fallback old-data values so audit persistence does not fail.

#### Scenario: Persist Add/Drop audit log with missing student financial data
- **GIVEN** an Add/Drop submission event for a student with null `StudentData`
- **WHEN** the audit payload is generated and saved
- **THEN** audit log persistence SHALL succeed
- **AND** `oldData.credits_taken` SHALL be `0.0`
- **AND** `oldData.remaining_capital` SHALL be `0`.

### Requirement: Bid points validation SHALL apply zero-capital fallback when StudentData is missing
The system SHALL evaluate Add/Drop bid point constraints using a fallback of `0` remaining capital when `StudentData` is null.

#### Scenario: Null StudentData with no new bid spend
- **GIVEN** Add/Drop validation input where `StudentData` is null and no new enrollment points are spent
- **WHEN** bid points validation runs
- **THEN** validation SHALL pass without runtime exception.

#### Scenario: Null StudentData with positive net spend
- **GIVEN** Add/Drop validation input where `StudentData` is null and total points to spend exceed total points refunded
- **WHEN** bid points validation runs
- **THEN** validation SHALL fail with an "Insufficient bid points" domain error.

### Requirement: Add/Drop validation SHALL reject duplicate courses in the same submission
The system SHALL reject Add/Drop enrollment payloads that contain more than one section of the same course in a single request.

#### Scenario: Duplicate course sections in one submission payload
- **GIVEN** an Add/Drop request with two enrollment rows that map to the same course but different class sections
- **WHEN** duplicate-in-submission validation runs
- **THEN** validation SHALL fail with a duplicate-course domain error.

### Requirement: Add/Drop validation SHALL reject duplicate courses against current campaign enrollment
The system SHALL reject Add/Drop enrollment requests when the student already has ENROLLED, SELECTED, or WAITLISTED status for the same course in the current campaign, unless that course is dropped in the same request.

#### Scenario: Add a course that is already enrolled or waitlisted in current campaign
- **GIVEN** the student already has an ENROLLED/SELECTED/WAITLISTED bid for a course in the current campaign
- **WHEN** the student submits a new enrollment for another section of that same course
- **THEN** validation SHALL fail with an explanatory duplicate-course domain error.

#### Scenario: Drop-then-add same course in one request
- **GIVEN** the student has an existing enrollment/waitlist for a course in the current campaign
- **AND** the same course is included in the drop list of the same Add/Drop request
- **WHEN** duplicate-with-current-enrollment validation runs
- **THEN** validation SHALL allow the add request to proceed for that course.
