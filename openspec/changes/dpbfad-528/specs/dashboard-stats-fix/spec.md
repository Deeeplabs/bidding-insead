## ADDED Requirements

### Requirement: Credit Progress uses Final Enrollment credit config
The Credit Progress metric SHALL use the credit range (`min_credits_per_student` / `max_credits_per_student`) from the Final Enrollment module's own phase config to determine whether a student is "on track". If the Final Enrollment phase config does not specify credit boundaries, the system SHALL fall back to the corresponding bidding round module's config.

#### Scenario: Final Enrollment has its own credit config
- **WHEN** a Final Enrollment module's phase config defines `min_credits_per_student` = 3.0 and `max_credits_per_student` = 5.0
- **AND** the corresponding bidding round config defines `min_credits_per_student` = 2.0 and `max_credits_per_student` = 6.0
- **THEN** the Credit Progress metric SHALL use 3.0–5.0 as the acceptable range for "on track" students in that Final Enrollment module

#### Scenario: Final Enrollment has no credit config — fallback to bidding round
- **WHEN** a Final Enrollment module's phase config does not define `min_credits_per_student` or `max_credits_per_student`
- **AND** the corresponding bidding round config defines `min_credits_per_student` = 2.0 and `max_credits_per_student` = 6.0
- **THEN** the Credit Progress metric SHALL use 2.0–6.0 from the bidding round config

### Requirement: Credit Progress aggregates across all active campaigns' Final Enrollment rounds
The total student count (denominator) for Credit Progress SHALL be the sum of eligible students across all Final Enrollment rounds of all active campaigns, regardless of whether those modules have any enrolled bids yet. The on-track count (numerator) SHALL be the count of those students whose enrolled credits fall within the applicable credit range.

#### Scenario: Multiple campaigns with Final Enrollment
- **WHEN** there are 3 active campaigns, each with 1 Final Enrollment round and 9 eligible students
- **THEN** Credit Progress total SHALL be 27
- **AND** the label SHALL read "{percentage}% students are on track (Totals across all active campaigns' Final Enrollment)"

#### Scenario: Final Enrollment module with no enrollments yet
- **WHEN** a Final Enrollment module exists but no students have enrolled bids
- **THEN** the eligible students for that FE module SHALL still be counted in the total
- **AND** on-track count for those students SHALL be 0

#### Scenario: No active campaigns with Final Enrollment
- **WHEN** no active campaigns have a Final Enrollment module
- **THEN** Credit Progress SHALL show 0 / 0 with 0% and the standard label text

### Requirement: Credit Progress uses consistent config format for eligibility
The config passed to the student eligibility service SHALL be in the same flattened format as used by `collectBiddingPhases()`: `moduleConfig['config']` merged with `student_selection`. This ensures student filters and selection criteria are correctly applied when counting eligible students.

### Requirement: Bidding Status Overview aggregates across all Pre-Bidding and Bidding rounds
The total student count (denominator) for Bidding Status Overview SHALL be the sum of eligible students across all Pre-Bidding and Bidding rounds of all active campaigns. A student counted in multiple rounds contributes to the total once per round.

#### Scenario: Multiple campaigns with multiple bidding rounds
- **WHEN** there are 3 active campaigns, each with 7 Pre-Bidding + Bidding rounds and 9 students each
- **THEN** Bidding Status total SHALL be 189 (3 × 7 × 9)
- **AND** the label SHALL read "{percentage}% students have done bidding (Totals across all active campaigns' Pre-Bidding and Bidding rounds)"

#### Scenario: No active campaigns
- **WHEN** there are no active campaigns
- **THEN** Bidding Status Overview SHALL show 0 / 0 with 0%
