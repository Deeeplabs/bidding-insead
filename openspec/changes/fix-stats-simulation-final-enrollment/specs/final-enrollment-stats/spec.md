## ADDED Requirements

### Requirement: Final Enrollment phase displays accurate student count
The system SHALL display the accurate total number of students enrolled in the Final Enrollment phase for a specific campaign and module.

#### Scenario: View final enrollment statistics after campaign creation
- **GIVEN** a campaign has been created and has progressed through bidding and simulation
- **WHEN** the Programme Manager navigates to the Final Enrollment phase detail view
- **THEN** the displayed student count SHALL match the actual number of students who completed enrollment

#### Scenario: View final enrollment statistics after campaign editing
- **GIVEN** a campaign exists with final enrollment statistics displayed
- **WHEN** the Programme Manager edits the campaign configuration
- **AND** returns to the Final Enrollment phase detail view
- **THEN** the displayed student count SHALL reflect the current state accurately

#### Scenario: Student count reflects final enrollment status
- **GIVEN** students have completed the enrollment process
- **WHEN** calculating the final enrollment student count
- **THEN** the count SHALL include all students who have successfully enrolled in courses

### Requirement: Final Enrollment phase displays accurate course count
The system SHALL display the accurate total number of courses/classes available in the Final Enrollment phase for a specific campaign and module.

#### Scenario: View course count in final enrollment
- **GIVEN** a campaign has reached the Final Enrollment phase
- **WHEN** the Programme Manager views the Final Enrollment statistics
- **THEN** the displayed course count SHALL match the number of classes in the campaign configuration

#### Scenario: Course count reflects final enrollment configuration
- **GIVEN** a campaign has been configured with courses for the Final Enrollment phase
- **WHEN** viewing final enrollment statistics
- **THEN** the course count SHALL reflect the courses defined in the CampaignModule for the Final Enrollment module

### Requirement: Statistics update correctly after campaign operations
The system SHALL ensure final enrollment statistics are consistent after campaign creation and modification operations.

#### Scenario: Statistics refresh after campaign save
- **GIVEN** a Programme Manager creates or edits a campaign
- **WHEN** the campaign is saved successfully
- **AND** the Programme Manager navigates to the Final Enrollment phase
- **THEN** the statistics SHALL load with accurate current data

#### Scenario: No stale data in final enrollment statistics
- **GIVEN** multiple campaign operations have been performed
- **WHEN** viewing final enrollment statistics
- **THEN** the displayed data SHALL reflect the most recent state, not cached outdated values
