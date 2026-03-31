## ADDED Requirements

### Requirement: Accurate Time Display for Student Portal
The Student Portal (`bidding-web`) SHALL display the correct local opening and closing times for all bidding rounds as configured in the backend, without any unintended timezone shifts (e.g., 8-hour offset).

#### Scenario: Round closing time display in SGT (GMT+8)
- **GIVEN** a bidding round is configured in the backend to close at `2026-03-27 17:20:00 UTC` (which is `2026-03-27 01:20:00 SGT`)
- **AND** a student's profile/browser timezone is set to `Asia/Singapore` (SGT)
- **WHEN** the student views the bidding round status card in the Student Portal
- **THEN** the closing time SHALL be displayed as `27 Mar 2026, 01:20 GMT+8`
- **AND** it SHALL NOT be displayed as `09:20 GMT+8`.

### Requirement: UTC Context Preservation in Frontend Date Parsing
The frontend components SHALL preserve the UTC context of date strings received from the API throughout the parsing and processing lifecycle, only converting to local time for final rendering to the user.

#### Scenario: Date parsing in CollapsePanelExtra
- **GIVEN** an `endDate` string `2026-03-27 17:20:00` (UTC) from the API
- **WHEN** the `CollapsePanelExtra` component processes this date
- **THEN** it SHALL parse it as `17:20:00 UTC`
- **AND** it SHALL pass the resulting `Date` object (or an ISO string with 'Z') to `getPhaseStatus` and `getPhaseDisplayText`
- **AND** it SHALL NOT convert it to an intermediate local-time-formatted string without timezone info before status calculation.

#### Scenario: Date parsing in HeaderSection
- **GIVEN** an `endDate` string from the API
- **WHEN** the `HeaderSection` component processes this date
- **THEN** it SHALL parse it as UTC
- **AND** it SHALL preserve this UTC context when calculating the countdown and display text.
