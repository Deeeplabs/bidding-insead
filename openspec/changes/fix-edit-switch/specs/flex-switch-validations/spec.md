## ADDED Requirements

### Requirement: Flex Switch Enrollment Validations
The system SHALL validate new flex switch target courses against the student's existing enrollments to prevent duplication.

#### Scenario: Switching into an already enrolled course
- **WHEN** a student attempts to switch into a course they are already enrolled in
- **THEN** the request MUST be rejected with a clear validation error message

### Requirement: Flex Switch Schedule Conflict Validations
The system SHALL prohibit a flex switch if the target class schedule directly conflicts with the student's active enrolled classes.

#### Scenario: Switching into a conflicting course
- **WHEN** a student attempts to switch to a course whose session/class timing conflicts directly with an existing active enrollment
- **THEN** the request MUST be rejected with a schedule conflict validation error

### Requirement: Flex Switch Rule Configuration Validation
The system SHALL ensure that bid points rule configurations are respected during a switch.

#### Scenario: Enforcing bid points
- **WHEN** a switch operation is processed
- **THEN** the required bid points validation MUST run according to the rules configuration defined for that campaign (e.g., deducting points or validating minimum required points, subject to PM's definition)
