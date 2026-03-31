## ADDED Requirements

### Requirement: Accurate Time Display for Student Portal
The Student Portal (`bidding-web`) SHALL display the correct local opening and closing times for all bidding rounds as configured in the backend, without any unintended timezone shifts (e.g., 8-hour offset or 1-hour offset).

#### Scenario: Round closing time display in SGT (GMT+8)
- **GIVEN** a bidding round is configured in the backend to close at `2026-03-27 17:20:00 UTC` (which is `2026-03-28 01:20:00 SGT`)
- **AND** a student's profile/browser timezone is set to `Asia/Singapore` (SGT)
- **WHEN** the student views the bidding round status card in the Student Portal
- **THEN** the closing time SHALL be displayed as `28 Mar 2026, 01:20 GMT+8`
- **AND** it SHALL NOT be displayed with any hour offset (e.g., `02:20 GMT+8` or `09:20 GMT+8`).

### Requirement: Configured Dates Match Displayed Dates
When both the PM and Student are in the same timezone (e.g., GMT+8), the start and end dates displayed on the PM Campaign list tooltip, the Student dashboard, and the pre-bidding module configuration form SHALL all show identical times.

#### Scenario: PM and Student in same timezone see matching dates
- **GIVEN** a PM configures a pre-bidding module with start date `2026-03-30 06:00` and end date `2026-04-04 05:00` (displayed in the PM's local time GMT+8)
- **AND** both the PM and Student browser timezones are GMT+8
- **WHEN** the PM views the Campaign list page tooltip for this module
- **AND** the Student views the Active Bidding Round dashboard
- **THEN** both SHALL display start as `30 Mar 2026, 06:00 GMT+8` and end as `4 Apr 2026, 05:00 GMT+8`
- **AND** the displayed times SHALL NOT differ from the configured times by any offset.

### Requirement: UTC Context Preservation in Backend DateTime Handling
All backend code paths that create, save, or compare DateTime objects SHALL use UTC explicitly, ensuring no server-local-timezone contamination.

#### Scenario: Phase activation stores UTC timestamps
- **GIVEN** a PM clicks "Start" on a pre-bidding phase at real-world time `2026-03-30 05:00 UTC`
- **WHEN** `CampaignPhaseService::activatePhase()` records the activation time
- **THEN** the `phase_config.start_date` DATETIME column SHALL store `2026-03-30 05:00:00` (UTC)
- **AND** the `module_config.actual_start_date` JSON field SHALL store `2026-03-30T05:00:00.000Z`
- **AND** it SHALL NOT store the Europe/Paris local time (e.g., `2026-03-30 07:00:00` CEST).

### Requirement: UTC Context Preservation in Frontend Date Parsing
The frontend components SHALL preserve the UTC context of date strings received from the API throughout the parsing and processing lifecycle, only converting to local time for final rendering to the user.

#### Scenario: Date parsing in CollapsePanelExtra
- **GIVEN** an `endDate` string `2026-03-27T17:20:00.000Z` from the API
- **WHEN** the `CollapsePanelExtra` component processes this date
- **THEN** it SHALL parse it as `17:20:00 UTC`
- **AND** it SHALL pass the resulting `Date` object to `getPhaseStatus` and `getPhaseDisplayText`
- **AND** it SHALL NOT convert it to an intermediate local-time-formatted string without timezone info before status calculation.

#### Scenario: Date parsing in HeaderSection
- **GIVEN** an `endDate` string from the API
- **WHEN** the `HeaderSection` component processes this date
- **THEN** it SHALL parse it as UTC
- **AND** it SHALL preserve this UTC context when calculating the countdown and display text.
