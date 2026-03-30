## ADDED Requirements

### Requirement: Bid submission SHALL validate total bid points against configured minimum capital
When a student submits a final bid (non-draft), the system SHALL reject the submission if the total bid points across all bids is below the configured `min_capital_per_student`. The system SHALL read the minimum exclusively from the `min_capital_per_student` config key, without falling back to count limits like `min_bids_entire_round`.

#### Scenario: Final bid submission rejected when total points below minimum capital
- **WHEN** a student submits final bids with total bid points of 50
- **AND** the campaign bidding round is configured with `min_capital_per_student` = 200
- **THEN** the system SHALL reject the submission with an error message indicating total bid points (50) is below minimum capital (200)

#### Scenario: Final bid submission accepted when total points meets minimum capital
- **WHEN** a student submits final bids with total bid points of 250
- **AND** the campaign bidding round is configured with `min_capital_per_student` = 200
- **THEN** the system SHALL accept the submission

#### Scenario: No minimum validation when config key is omitted
- **WHEN** a student submits final bids with total bid points of 5
- **AND** the campaign bidding round has `min_capital_per_student` = null, even if `min_bids_entire_round` is set
- **THEN** the system SHALL accept the submission (validations strictly use matching metric types)

#### Scenario: Draft bid submission bypasses minimum capital check
- **WHEN** a student saves bids as draft with total bid points of 10
- **AND** the campaign bidding round is configured with `min_capital_per_student` = 200
- **THEN** the system SHALL allow the draft save (validation only applies to final submissions)

### Requirement: Bid submission SHALL validate total bid points against configured maximum capital
The existing maximum capital validation SHALL continue to use the `max_capital_per_student` config key. This requirement documents the existing behavior for completeness.

#### Scenario: Final bid submission rejected when total points exceeds maximum capital
- **WHEN** a student submits final bids with total bid points of 1500
- **AND** the campaign bidding round is configured with `max_capital_per_student` = 1000
- **THEN** the system SHALL reject the submission with an error indicating total bid points (1500) exceeds available capital (1000)

### Requirement: Generate Bid Data SHALL produce bids within configured capital range
When an admin generates dummy bid data via the Generate Bid Data function, the total bid points per student SHALL fall within the configured `min_capital_per_student` and `max_capital_per_student` range, without relying on block minimum counts as fallbacks.

#### Scenario: Generated bid points respect minimum capital
- **WHEN** an admin generates bid data for a campaign
- **AND** the bidding round is configured with `min_capital_per_student` = 200
- **THEN** every student's total generated bid points SHALL be at least 200

#### Scenario: Generated bid points respect maximum capital
- **WHEN** an admin generates bid data for a campaign
- **AND** the bidding round is configured with `max_capital_per_student` = 1000
- **THEN** every student's total generated bid points SHALL NOT exceed 1000

#### Scenario: Minimum enforcement spread across generated bids
- **WHEN** the bid generation logic computes a shortfall of 80 points to reach `min_capital_per_student`
- **AND** multiple valid bids were assigned 
- **THEN** the system SHALL incrementally spread the shortfall points (up to 50 at a time) across all available bids iteratively until the shortfall reaches 0 to produce a normalized dataset.
