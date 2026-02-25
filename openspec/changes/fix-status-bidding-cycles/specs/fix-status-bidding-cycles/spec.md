## ADDED Requirements

### Requirement: Main Campaign Updates Must NOT Reset Phase Statuses
The system SHALL ensure that when the main campaign configuration (e.g., `min_total_points`, `max_credits_default`) is updated via the API, the save operation isolates state mutations exclusively to the main campaign. It MUST NOT cascade or defensively alter the status of its child phases, such as resetting an active Bidding round to a "Not Started" state.

#### Scenario: Updating a campaign during active bidding
- **GIVEN** a live campaign with an "Active" Bidding phase and enrolled bids
- **WHEN** an administrator submits an update to the top-level Campaign configuration (e.g., changes minimum total points via the Edit screen)
- **THEN** the Bidding phase status SHALL persistently remain "Active"
- **AND** the bids and statistical counters SHALL remain intact and visible.

### Requirement: Explicit Phase State Isolation
The system SHALL only transition a Bidding cycle from "Active" to "Closed" or "Not Started" when an administrator explicitly targets the Bidding phase directly, not when making unrelated campaign-level config changes.

#### Scenario: Phase coercion prevention via campaign updates
- **GIVEN** an API request updating non-phase-specific properties on the main campaign
- **WHEN** the campaign update service processes the request
- **THEN** it SHALL NOT clear, coerce, or reset the ongoing phase statuses.

### Requirement: Simulation Progression Gate
The system SHALL correctly honor the simulation phase boundary, ensuring that clicking into the Simulation module does not erroneously skip to "Final Enrollment" due to corrupted state values.

#### Scenario: Accessing pending Simulation
- **GIVEN** the Bidding phase has ended and Simulation is correctly initialized
- **WHEN** the administrator attempts to load the Simulation layout
- **THEN** the system SHALL load the Simulation dashboard and NOT jump immediately to the Final Enrollment screen.
