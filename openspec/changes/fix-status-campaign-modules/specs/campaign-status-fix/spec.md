## ADDED Requirements

### Requirement: Simulation Handles Null Dates Correctly

The SimulationDashboardService SHALL properly handle missing/null dates when calculating phase status, defaulting to "Not Started" instead of "Completed".

#### Scenario: Phase with null startDate
- **WHEN** getModulePhaseDetails() is called and startDate is null
- **THEN** the system SHALL set the phase status to CLOSE (Not Started)
- **AND** the UI SHALL allow running simulation, rebalancing, and other operations

#### Scenario: Phase with null endDate
- **WHEN** getModulePhaseDetails() is called and endDate is null
- **THEN** the system SHALL set the phase status to CLOSE (Not Started)
- **AND** the simulation UI SHALL NOT be locked

#### Scenario: Phase with valid dates
- **WHEN** getModulePhaseDetails() is called and both startDate and endDate are set
- **THEN** the system SHALL calculate status based on current date vs. phase dates
- **AND** existing behavior SHALL be preserved

### Requirement: Campaign Duplication Maintains Reference Integrity

The CampaignDuplicationService SHALL explicitly link new CampaignPhaseConfig records to their CampaignModule during duplication.

#### Scenario: Duplicate campaign creates phase configs
- **WHEN** CampaignDuplicationService::duplicateCampaign() creates new CampaignPhaseConfig objects
- **THEN** each new phase config SHALL have its campaign_module_id set to the new CampaignModule's ID
- **AND** the relationship SHALL be persisted to the database

#### Scenario: View duplicated campaign details
- **WHEN** PM views the detail bidding page for a duplicated campaign
- **THEN** the system SHALL successfully resolve all module references
- **AND** no "undefined status" errors SHALL occur

### Requirement: Module Activation on Configuration Save

The CampaignPhaseService SHALL set isActive=true when PM saves configuration or dates for a duplicated module.

#### Scenario: Save module configuration
- **WHEN** PM saves module configuration via saveModuleConfigByCampaignAndModule()
- **THEN** the system SHALL set isActive=true on the CampaignPhaseConfig
- **AND** the module SHALL be accessible via the API

#### Scenario: Configure add/drop deadline
- **WHEN** PM configures add/drop deadline via configureAddDropDeadline()
- **THEN** the system SHALL set isActive=true on the CampaignPhaseConfig
- **AND** subsequent requests to load module details SHALL succeed

#### Scenario: Load module details after activation
- **WHEN** PM attempts to load bidding details for a module that was just configured
- **THEN** the API SHALL return the module data (not 404)
- **AND** the module SHALL be marked as active

### Requirement: Auto-Activate Phase Config for Existing Campaigns

The AdminCampaignDetailService SHALL auto-activate phase configs that have dates set but isActive=false when accessing the detail endpoint.

#### Scenario: Access module detail for existing duplicated campaign
- **WHEN** PM accesses module detail for a campaign that was duplicated BEFORE the fix was deployed
- **THEN** the system SHALL auto-activate phase configs that have dates but isActive=false
- **AND** the API SHALL return the module data (not 404)

#### Scenario: Access new duplicated campaign
- **WHEN** PM accesses module detail for a campaign that was duplicated AFTER the fix was deployed
- **THEN** the system SHALL use the existing isActive=true flag
- **AND** no auto-activation SHALL be needed

### Requirement: Backend API Returns Correct Status

The backend API SHALL return accurate status information that reflects the actual state of phases.

#### Scenario: GET simulation status with null dates
- **WHEN** frontend requests simulation status and dates are null
- **THEN** the API SHALL return "Not Started" status
- **AND** all operation buttons SHALL be enabled

#### Scenario: GET campaign detail after duplication
- **WHEN** frontend requests campaign detail after duplication
- **THEN** the API SHALL return valid status for all modules
- **AND** no undefined/null references SHALL be present

## ADDED Requirements: Secondary Issue - Simulation Auto-Run

### Requirement: Simulation Manual Trigger

The system SHALL require manual intervention to run simulation; it SHALL NOT automatically run simulation when PM closes the bidding round.

#### Scenario: Close bidding round manually
- **WHEN** PM clicks "Close Bidding" or similar action to end the bidding phase
- **THEN** the system SHALL NOT automatically start simulation
- **AND** the system SHALL wait for PM to explicitly trigger simulation

#### Scenario: Manual simulation trigger
- **WHEN** PM clicks "Run Simulation" or similar action
- **THEN** the system SHALL execute the simulation
- **AND** after completion, the UI SHALL allow Re-run, Commit, Rollback, Export actions
