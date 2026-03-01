# Student Dashboard Stats

## ADDED Requirements

### Requirement: Capital Left must never be negative (Dashboard)

The system SHALL ensure that `capital_left` returned by the dashboard stats endpoint is never a negative value.

#### Scenario: Capital left with overspending
- **WHEN** a student's total spent capital (200) exceeds their granted capital from campaigns (150)
- **THEN** the `capital_left` field SHALL be 0 (not -50)

#### Scenario: Capital left from single campaign
- **WHEN** a student participates in one campaign with minCapitalGranted of 500 and has spent 200
- **THEN** the `capital_left` field SHALL be 300 (500 + adjustments - 200)

#### Scenario: Capital left from multiple campaigns
- **WHEN** a student participated in Campaign A (minCapitalGranted: 500) and Campaign B (minCapitalGranted: 300), spent 400 total
- **THEN** the `capital_left` field SHALL be 400 (500 + 300 + adjustments - 400)

#### Scenario: Capital left with manual adjustments
- **WHEN** total minCapitalGranted is 500, manual adjustment is +100, spent is 200
- **THEN** the `capital_left` field SHALL be 400 (500 + 100 - 200)

### Requirement: Capital Left must never be negative (Bidding Round View)

The system SHALL ensure that `capital_left` in the active bidding round module view is never a negative value.

#### Scenario: Per-module capital overspending
- **WHEN** a student has bid 1200 cumulative points in a bidding round module with capitalGranted of 1000
- **THEN** the module's `capital_left` field SHALL be 0 (not -200)

#### Scenario: Normal per-module capital spending
- **WHEN** a student has bid 400 cumulative points in a bidding round module with capitalGranted of 1000
- **THEN** the module's `capital_left` field SHALL be 600

### Requirement: Credits to be fulfilled must never be negative

The system SHALL calculate `credits_to_be_fulfilled` as the accumulation of `minCreditsToFulfill` from all campaigns the student has participated in, minus credits earned, clamped to a minimum of 0.

#### Scenario: Credits to be fulfilled from single campaign
- **WHEN** a student participates in one campaign with minCreditsToFulfill of 10 and has earned 4 credits
- **THEN** the `credits_to_be_fulfilled` field SHALL be 6 (10 - 4)

#### Scenario: Credits to be fulfilled with overfulfillment
- **WHEN** a student has earned 15 credits but only needs 10 from campaigns
- **THEN** the `credits_to_be_fulfilled` field SHALL be 0 (not -5)

#### Scenario: Credits to be fulfilled from multiple campaigns
- **WHEN** a student participated in Campaign A (minCreditsToFulfill: 10) and Campaign B (minCreditsToFulfill: 8), earned 12 total
- **THEN** the `credits_to_be_fulfilled` field SHALL be 6 (10 + 8 - 12)

### Requirement: Credits left must use proper calculation

The system SHALL calculate the `credits.left` value in `StudentCreditService::getCredits()` as `max(0, totalCredits - creditsEarned)` instead of a hardcoded value.

#### Scenario: Credits left calculation
- **WHEN** a student has totalCredits of 10 and has earned 4 credits
- **THEN** the credits `left` field SHALL be 6

#### Scenario: Credits left with over-earning
- **WHEN** a student has totalCredits of 10 and has earned 12 credits
- **THEN** the credits `left` field SHALL be 0 (not -2)

### Requirement: Find campaigns via bid placement

The system SHALL identify which campaigns a student has participated in by finding campaigns where the student has placed bids.

#### Scenario: Student with bids in one campaign
- **WHEN** a student has placed bids in Campaign A only
- **THEN** only Campaign A's minCapitalGranted and minCreditsToFulfill SHALL be included in calculations

#### Scenario: Student with no bids
- **WHEN** a student has not placed any bids
- **THEN** capital_left and credits_to_be_fulfilled SHALL both be 0

### Requirement: Backward compatibility

The system SHALL maintain backward compatibility with existing frontend API consumers.

#### Scenario: API response fields preserved
- **WHEN** frontend calls the stats endpoint
- **THEN** the response SHALL include: capital_spent, capital_left, credits_earned, credits_to_be_fulfilled

#### Scenario: Existing frontend works without changes
- **WHEN** existing bidding-web dashboard calls the stats endpoint
- **THEN** it SHALL receive the same field names and display correctly

#### Scenario: Values are always non-negative
- **WHEN** any frontend consumer receives the API response
- **THEN** capital_left, credits_earned, and credits_to_be_fulfilled SHALL all be >= 0
