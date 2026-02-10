## ADDED Requirements

### Requirement: Duplicated campaign modules start as CLOSE
When a campaign is duplicated, all modules in the new campaign SHALL have `is_active = 'CLOSE'` regardless of the source campaign's module statuses or date ranges.

#### Scenario: Duplicate campaign with active modules in source
- **WHEN** a PM duplicates a campaign that has modules with `is_active = 'OPEN'` (dates spanning current date)
- **THEN** all modules in the duplicated campaign SHALL have `is_active = 'CLOSE'`

#### Scenario: Duplicate campaign with completed modules in source
- **WHEN** a PM duplicates a campaign that has modules with `is_active = 'COMPLETED'` (past date ranges)
- **THEN** all modules in the duplicated campaign SHALL have `is_active = 'CLOSE'`

#### Scenario: Duplicate with bidding rounds enabled preserves dates but not active state
- **WHEN** a PM duplicates a campaign with "Duplicate all configurations of Bidding Rounds" enabled
- **THEN** the `CampaignPhaseConfig` records SHALL be created with `isActive = false`
- **AND** the date ranges (start_date, end_date) SHALL be preserved from the source
- **AND** module config SHALL be preserved from the source

#### Scenario: Duplicate without bidding rounds
- **WHEN** a PM duplicates a campaign with "Duplicate all configurations of Bidding Rounds" disabled
- **THEN** no `CampaignPhaseConfig` records SHALL be created
- **AND** modules SHALL have `is_active = 'CLOSE'`

### Requirement: Draft campaigns enforce CLOSE status on all modules
Any campaign in Draft status SHALL report `is_active = 'CLOSE'` for all modules in API responses, regardless of `CampaignPhaseConfig` date ranges or `isActive` flags.

#### Scenario: Draft campaign with phase config dates spanning current date
- **WHEN** the API returns module data for a Draft campaign
- **AND** the campaign has phase configs with start_date <= now <= end_date
- **THEN** all modules SHALL have `is_active = 'CLOSE'`

#### Scenario: Draft campaign module status in admin API
- **WHEN** the admin dashboard requests campaign details for a Draft campaign
- **THEN** all campaign modules SHALL have `is_active = 'CLOSE'`
- **AND** all phase configs SHALL have `is_active = 'CLOSE'`

#### Scenario: Draft campaign module status in student API
- **WHEN** the student portal requests campaign module details for a Draft campaign
- **THEN** all modules SHALL have `is_active = false` (boolean, student-facing mapper)

#### Scenario: Non-Draft campaign module status is unaffected
- **WHEN** the API returns module data for a campaign that is NOT in Draft status
- **THEN** module `is_active` SHALL continue to be computed based on date ranges as before

### Requirement: Auto-transition campaign from draft to open when PM starts a module
When a PM opens any module phase via `PUT /dashboard/campaign/status` (`CampaignPhaseService::activatePhase()`), and the campaign is in `draft` status, the campaign status SHALL automatically transition to `open`.

#### Scenario: PM opens first module on a draft campaign
- **WHEN** the PM calls `activatePhase` with `status = 'open'` for a phase config
- **AND** the campaign is in `draft` status
- **THEN** the campaign status SHALL be set to `open`
- **AND** the phase `startDate` SHALL be set to now
- **AND** the API response for modules SHALL reflect the newly opened phase as `is_active = 'OPEN'` (since the campaign is no longer draft)

#### Scenario: PM opens module on an already-open campaign
- **WHEN** the PM calls `activatePhase` with `status = 'open'` for a phase config
- **AND** the campaign is already in `open` status
- **THEN** the campaign status SHALL remain `open` (no change)
- **AND** the phase SHALL be activated normally

#### Scenario: PM opens module on a closed campaign
- **WHEN** the PM calls `activatePhase` with `status = 'open'` for a phase config
- **AND** the campaign is in `close` status
- **THEN** the campaign status SHALL NOT be changed (remains `close`)
- **AND** the phase SHALL be activated normally

#### Scenario: PM closes a module (does not affect campaign status)
- **WHEN** the PM calls `activatePhase` with `status = 'close'` for a phase config
- **THEN** the campaign status SHALL NOT change regardless of the campaign's current status

### Requirement: Campaign status transition resets module display
When a campaign transitions to Draft status, the module status display SHALL immediately reflect CLOSE for all modules without requiring manual intervention.

#### Scenario: Campaign reverted to Draft shows all modules as CLOSE
- **WHEN** a campaign that was previously Open is set back to Draft status
- **THEN** all modules SHALL display as `is_active = 'CLOSE'` in the next API response
