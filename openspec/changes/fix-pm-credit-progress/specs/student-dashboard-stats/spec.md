## ADDED Requirements

### Requirement: Update Dashboard Stats Calculations
The system SHALL accurately calculate both student and PM dashboard progress metrics based on the active bidding cycle and PM configurations, rather than partial historical or static data.

#### Scenario: Student checks dashboard credits and capital
- **WHEN** the student dashboard API or PM student list/summary API is evaluated for stats
- **THEN** it SHALL calculate Credit Earned as the sum of Student Final Enrollment Courses Credits and adjustment credits for 2 bidding rounds
- **AND** it SHALL calculate Credits to be fulfilled as the sum of minimum credits required per student for the entire bidding cycle configured by PM
- **AND** it SHALL calculate Capital Spent as the sum of bid points spent in 2 bidding rounds
- **AND** it SHALL calculate Capital Left as the Total Capital granted to the student for the entire campaign minus Capital Spent
- **AND** the values displayed on the PM Dashboard MUST match the exact identical ruleset as the Student Dashboard

#### Scenario: Accurate Capital and Credits across applications
- **WHEN** the PM accesses `/api/admin/student/{id}` or the `StudentListDto`
- **THEN** it SHALL accurately reflect the updated outputs from `StudentCapitalService` and `StudentCreditService` paritying the exact rules matching the `student-dashboard-stats` configuration.
