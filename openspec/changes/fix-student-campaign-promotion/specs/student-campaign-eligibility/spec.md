## MODIFIED Requirements

### Requirement: Cross-promotion campaign discovery via JSON-stored student includes

When a student from a different promotion is added to a campaign via per-module configuration (stored in `module_config.config.student_filters` JSON), the system SHALL discover and include that campaign in the student's active bidding rounds, in addition to existing table-based discovery.

#### Scenario: Student added via module config JSON sees campaign (cross-promotion discovery)
- **GIVEN** a campaign exists for promotion A with Pre-bidding and Bidding Round modules
- **AND** admin adds student X (from promotion B) to the Pre-bidding module via module configuration
- **AND** the `student_include` filter is stored in `CampaignPhaseConfig.module_config.config.student_filters` JSON (NOT in `campaign_student_filters` table)
- **AND** the campaign status is "open" and current date is within the campaign's date range
- **WHEN** student X requests their active bidding rounds via the student dashboard
- **THEN** the system SHALL discover the campaign through JSON-based cross-promotion lookup
- **AND** the campaign SHALL appear in the student's active bidding rounds

#### Scenario: Student added via DB table filter sees campaign (existing behavior preserved)
- **GIVEN** a campaign exists for promotion A
- **AND** admin adds student X (from promotion B) via campaign-level filter (stored in `campaign_student_filters` table)
- **AND** the campaign status is "open"
- **WHEN** student X requests their active bidding rounds
- **THEN** the system SHALL discover the campaign through `findOpenCampaignsByStudentInclude` (table-based lookup)
- **AND** existing behavior is unchanged

#### Scenario: Student added to pre-bidding only sees campaign during pre-bidding phase
- **GIVEN** a campaign has Pre-bidding (Jan 1-15) and Bidding Round (Jan 16-31) phases
- **AND** student X is added via Pre-bidding module config with `student_include`
- **AND** current date is Jan 10 (within Pre-bidding phase)
- **WHEN** student X requests their active bidding rounds
- **THEN** the campaign SHALL be included in the response
- **AND** `isStudentEligibleForCampaign` SHALL pass because the merged filters contain the `student_include`

#### Scenario: Student added to pre-bidding only does NOT see campaign during bidding round
- **GIVEN** a campaign has Pre-bidding (Jan 1-15) and Bidding Round (Jan 16-31) phases
- **AND** student X is added via Pre-bidding module config with `student_include`
- **AND** current date is Jan 20 (within Bidding Round phase)
- **WHEN** student X requests their active bidding rounds
- **THEN** the campaign is discovered by JSON search (student ID is still in JSON)
- **BUT** `isStudentEligibleForCampaign` SHALL return false because:
  - The active phase is Bidding Round
  - `configFilters` comes from Bidding Round's config (which does NOT contain student X's include)
  - `tableFilters` has no matching phase filter for student X
  - The merged filters do not include student X
- **AND** the campaign SHALL NOT appear in the response

#### Scenario: Student from same promotion sees campaign normally
- **GIVEN** a campaign exists for promotion X
- **AND** student A belongs to promotion X
- **AND** the campaign status is "open"
- **WHEN** student A requests their active bidding rounds
- **THEN** the system SHALL include the campaign via promotion-based query (existing behavior unchanged)

#### Scenario: De-duplicated campaign list when student found via multiple paths
- **GIVEN** a campaign exists for promotion A
- **AND** student X is found via both table-based AND JSON-based cross-promotion lookup
- **WHEN** student X requests their active bidding rounds
- **THEN** the campaign SHALL appear only ONCE in the response
- **AND** duplicate detection SHALL use campaign ID comparison

#### Scenario: Campaign with no active phase is excluded
- **GIVEN** a campaign has modules configured
- **AND** student X is included via module config JSON
- **AND** no phase has start/end dates encompassing the current date
- **WHEN** student X requests their active bidding rounds
- **THEN** the campaign SHALL NOT appear (no active phase)
