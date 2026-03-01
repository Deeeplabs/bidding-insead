## ADDED Requirements

### Requirement: Simulation phase displays accurate student count
The system SHALL display the accurate total number of students who participated in the simulation phase for a specific campaign and module.

#### Scenario: View simulation statistics after campaign creation
- **GIVEN** a campaign has been created with students who submitted bids
- **WHEN** the Programme Manager navigates to the Simulation phase detail view
- **THEN** the displayed student count SHALL match the actual number of students who submitted bids in the associated bidding round

#### Scenario: View simulation statistics after campaign editing
- **GIVEN** a campaign exists with simulation statistics displayed
- **WHEN** the Programme Manager edits the campaign (e.g., adds/removes modules, adjusts timing)
- **AND** returns to the Simulation phase detail view
- **THEN** the displayed student count SHALL reflect the current state accurately

#### Scenario: Student count excludes invalid bids
- **GIVEN** students have submitted bids in the bidding round
- **WHEN** calculating the simulation student count
- **THEN** the count SHALL only include bids with valid status (e.g., not cancelled, not expired)

### Requirement: Simulation phase displays accurate course count
The system SHALL display the accurate total number of courses/classes available in the simulation phase for a specific campaign and module.

#### Scenario: View course count after simulation run
- **GIVEN** a simulation has been run for a campaign
- **WHEN** the Programme Manager views the Simulation phase statistics
- **THEN** the displayed course count SHALL match the number of classes that were included in the simulation

#### Scenario: Course count reflects campaign configuration
- **GIVEN** a campaign has been configured with specific courses for the bidding round
- **WHEN** viewing simulation statistics
- **THEN** the course count SHALL reflect the courses defined in the CampaignModule configuration

### Requirement: Statistics update correctly after campaign operations
The system SHALL ensure simulation statistics are consistent after campaign creation and modification operations.

#### Scenario: Statistics refresh after campaign save
- **GIVEN** a Programme Manager creates or edits a campaign
- **WHEN** the campaign is saved successfully
- **AND** the Programme Manager navigates to the Simulation phase
- **THEN** the statistics SHALL load with accurate current data

#### Scenario: No stale data in simulation statistics
- **GIVEN** multiple campaign operations have been performed
- **WHEN** viewing simulation statistics
- **THEN** the displayed data SHALL reflect the most recent state, not cached outdated values
