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

### Requirement: Campaign status transition resets module display
When a campaign transitions to Draft status, the module status display SHALL immediately reflect CLOSE for all modules without requiring manual intervention.

#### Scenario: Campaign reverted to Draft shows all modules as CLOSE
- **WHEN** a campaign that was previously Open is set back to Draft status
- **THEN** all modules SHALL display as `is_active = 'CLOSE'` in the next API response
