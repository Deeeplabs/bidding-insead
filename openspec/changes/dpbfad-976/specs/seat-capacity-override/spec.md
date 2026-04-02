## ADDED Requirements

### Requirement: Student-Facing Seat Capacity Must Reflect PM Bidding Configuration

All student-facing views and validations SHALL use the PM-configured seat capacity from `AdjustmentCourse.seatCapacity` (when present) instead of the default `ClassPromotions.promotionSeats` sum. The fallback chain is: `SimulationAdjustmentCourse.seatCapacity` → `AdjustmentCourse.seatCapacity` → `Class.getTotalPromotionSeats()`.

#### Scenario: Bidding round detail shows configured seat capacity after bid submission
- **GIVEN** a PM has created a campaign with a bidding module
- **AND** the PM has set the seat capacity for all courses to "3" in the bidding configuration (stored as `AdjustmentCourse` records)
- **AND** the default `ClassPromotions.promotionSeats` for those classes is "90"
- **WHEN** a student submits a bid and views the bidding round detail page
- **THEN** the `total_seats` field for each course SHALL be "3" (from `AdjustmentCourse`)
- **AND** it SHALL NOT revert to "90" (the default `ClassPromotions` value)

#### Scenario: Active bidding rounds list shows configured seat capacity
- **GIVEN** a PM has set seat capacity to "3" via `AdjustmentCourse` for a campaign
- **WHEN** a student views the active bidding rounds list
- **THEN** all seat capacity values SHALL reflect the configured "3"
- **AND** available seats SHALL be calculated as `3 - (enrolled + invited + waitlisted)`

#### Scenario: Bid validation uses configured seat capacity
- **GIVEN** a PM has set seat capacity to "3" via `AdjustmentCourse`
- **AND** 3 students are already enrolled in a class
- **WHEN** a 4th student attempts to bid on that class
- **THEN** the validation SHALL treat the class as full (capacity = 3, not 90)

#### Scenario: Add/drop validation uses configured seat capacity
- **GIVEN** a PM has set seat capacity to "3" via `AdjustmentCourse`
- **AND** 3 students are enrolled in a class
- **WHEN** a student attempts to add-drop into that class
- **THEN** the validator SHALL treat the class as full based on the configured capacity of "3"
- **AND** the student SHALL be placed on the waitlist (if waitlist is enabled)

#### Scenario: Waitlist capacity check uses configured seat capacity
- **GIVEN** a PM has set seat capacity to "3" via `AdjustmentCourse`
- **AND** 2 students are enrolled in a class
- **WHEN** the waitlist service checks available seats
- **THEN** it SHALL calculate available seats as `3 - 2 = 1`
- **AND** it SHALL NOT calculate as `90 - 2 = 88`

#### Scenario: Waitlist auto-offer uses configured seat capacity
- **GIVEN** a PM has set seat capacity to "3" via `AdjustmentCourse`
- **AND** 1 enrolled student drops the class (enrolled becomes 2)
- **WHEN** the auto-offer service checks for available seats
- **THEN** it SHALL determine there is 1 available seat (capacity 3 - 2 enrolled)
- **AND** it SHALL offer the seat to the next waitlisted student

#### Scenario: No adjustment configured falls back to default
- **GIVEN** a PM has NOT created any `AdjustmentCourse` records for a campaign
- **WHEN** a student views seat capacity for any class
- **THEN** `total_seats` SHALL equal `Class.getTotalPromotionSeats()` (sum of `ClassPromotions.promotionSeats`)
- **AND** behavior SHALL be identical to the current default

#### Scenario: PM view remains unchanged
- **GIVEN** a PM has set seat capacity to "3" via `AdjustmentCourse`
- **WHEN** the PM views the campaign course list or dashboard
- **THEN** the seat capacity SHALL continue to show "3"
- **AND** no change in PM-facing behavior SHALL occur

## MODIFIED Requirements

### Requirement: Seat Capacity Fallback Chain Consistency
All code paths that read seat capacity SHALL use a consistent fallback chain: `SimulationAdjustmentCourse.seatCapacity` → `AdjustmentCourse.seatCapacity` → `Class.getTotalPromotionSeats()`. This applies to display, validation, and business logic uniformly.
