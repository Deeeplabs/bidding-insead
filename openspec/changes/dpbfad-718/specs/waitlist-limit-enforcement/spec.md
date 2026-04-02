## ADDED Requirements

### Requirement: Per-student waitlist limit is enforced during Add/Drop submission
When a student submits an Add/Drop request, the system SHALL validate that the total number of waitlisted courses does not exceed the PM-configured per-student waitlist limit.

#### Scenario: Student adds courses that would exceed the waitlist limit
- **GIVEN** the PM configured `waitlist_capacity = 3` (or `max_waitlist_slots_per_student = 3`) in the Add/Drop module config
- **AND** the student currently has 2 waitlisted courses
- **WHEN** the student submits an Add/Drop request with 2 new enrollments that would both become waitlisted (classes are full)
- **THEN** the system SHALL reject the submission with an error message indicating the waitlist limit would be exceeded
- **AND** no bids SHALL be created or modified

#### Scenario: Student adds courses within the waitlist limit
- **GIVEN** the PM configured `waitlist_capacity = 5` in the Add/Drop module config
- **AND** the student currently has 2 waitlisted courses
- **WHEN** the student submits an Add/Drop request with 2 new enrollments that would become waitlisted
- **THEN** the system SHALL accept the submission (total = 4, within limit of 5)

#### Scenario: Waitlist limit not configured — no enforcement
- **GIVEN** the Add/Drop module config does NOT contain `max_waitlist_slots_per_student` or `waitlist_capacity`
- **WHEN** the student submits an Add/Drop request
- **THEN** no per-student waitlist limit SHALL be enforced (unlimited)

#### Scenario: Waitlist capacity toggle is disabled — no enforcement
- **GIVEN** `limit_waitlist_capacity = false` in the Add/Drop module config
- **WHEN** the student submits an Add/Drop request
- **THEN** no per-student waitlist limit SHALL be enforced regardless of `waitlist_capacity` value

## MODIFIED Requirements

### Requirement: max_waitlist_allowed displays the PM-configured value
The student-facing API response SHALL return `max_waitlist_allowed` matching the value configured by the PM in the Add/Drop & Waitlist module.

#### Scenario: PM configured waitlist_capacity = 5 in Add/Drop module
- **GIVEN** the PM set `limit_waitlist_capacity = true` and `waitlist_capacity = 5` in the Add/Drop module config
- **WHEN** the student views the Add/Drop page
- **THEN** `max_waitlist_allowed` SHALL be `5`

#### Scenario: PM configured max_waitlist_slots_per_student = 3 in Add/Drop module
- **GIVEN** the PM set `max_waitlist_slots_per_student = 3` in the Add/Drop module config
- **WHEN** the student views the Add/Drop page
- **THEN** `max_waitlist_allowed` SHALL be `3`

#### Scenario: Both keys present — max_waitlist_slots_per_student takes precedence
- **GIVEN** the config contains both `max_waitlist_slots_per_student = 4` and `waitlist_capacity = 5`
- **WHEN** the student views the Add/Drop page
- **THEN** `max_waitlist_allowed` SHALL be `4` (canonical key takes precedence)

#### Scenario: Neither key present — default to 0
- **GIVEN** the Add/Drop module config contains neither `max_waitlist_slots_per_student` nor `waitlist_capacity`
- **WHEN** the student views the Add/Drop page
- **THEN** `max_waitlist_allowed` SHALL be `0` (no limit, no enforcement)

### Requirement: EnrollmentViewService reads waitlist limit from correct module
`EnrollmentViewService::getMaxWaitlistAllowed()` SHALL read the per-student waitlist limit from the `add_drop_waitlist` module config, NOT from `final_enrollment`.

#### Scenario: Add/Drop module has waitlist_capacity configured
- **GIVEN** the `add_drop_waitlist` module has `waitlist_capacity = 5` in its config
- **AND** the `final_enrollment` module has no waitlist config
- **WHEN** the enrollment view is rendered
- **THEN** `max_waitlist_allowed` SHALL be `5`

#### Scenario: No Add/Drop module exists — fallback to 0
- **GIVEN** the campaign has no `add_drop_waitlist` module
- **WHEN** the enrollment view is rendered
- **THEN** `max_waitlist_allowed` SHALL be `0`

### Requirement: CampaignWaitlistService reads correct config keys
`CampaignWaitlistService::getMaxWaitlistPerStudent()` SHALL check `max_waitlist_slots_per_student`, then fall back to `waitlist_capacity`, then `max_waitlist_per_student` for backward compatibility.

#### Scenario: Config has waitlist_capacity only
- **GIVEN** the active phase config contains `waitlist_capacity = 5` but no `max_waitlist_slots_per_student`
- **WHEN** the waitlist service reads the per-student limit
- **THEN** the limit SHALL be `5`

### Requirement: Admin UI sends canonical config key
When the PM saves Add/Drop & Waitlist configuration, the admin UI SHALL send `max_waitlist_slots_per_student` alongside `waitlist_capacity` to ensure both legacy and canonical keys are present.

#### Scenario: PM saves Add/Drop config with waitlist capacity enabled
- **GIVEN** the PM enables waitlist capacity and sets the limit to 5
- **WHEN** the configuration is saved
- **THEN** the API payload SHALL include both `waitlist_capacity: 5` and `max_waitlist_slots_per_student: 5`

#### Scenario: PM saves Add/Drop config with waitlist capacity disabled
- **GIVEN** the PM disables waitlist capacity
- **WHEN** the configuration is saved
- **THEN** `max_waitlist_slots_per_student` SHALL NOT be included in the payload (or be null)
