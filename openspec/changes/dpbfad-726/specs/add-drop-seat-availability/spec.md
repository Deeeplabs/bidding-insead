## ADDED Requirements

### Requirement: Available courses with seats SHALL be discoverable
The system SHALL return all course-sections with available seats in the add/drop available courses endpoint, regardless of whether `ClassPromotions` records are partially misconfigured. When seat capacity is determined via `SimulationAdjustmentCourse` or `AdjustmentCourse` overrides, the course SHALL be included even if `ClassPromotions`-based seat calculation returns zero. The system SHALL log a warning when falling back to override-based seat counts.

#### Scenario: Course with seats available via ClassPromotions is returned
- **WHEN** a student requests available courses during the add/drop phase
- **AND** a class has `ClassPromotions.promotionSeats > 0` matching the student's promotion
- **AND** enrolled count is less than total seats
- **THEN** the course-section SHALL appear in the available courses list with status `available`

#### Scenario: Course with seats available via SimulationAdjustmentCourse override is returned
- **WHEN** a student requests available courses during the add/drop phase
- **AND** a class has no matching `ClassPromotions` record for the student's promotion
- **AND** a `SimulationAdjustmentCourse` record exists with `seatCapacity > enrolledCount`
- **THEN** the course-section SHALL appear in the available courses list with status `available`
- **AND** the system SHALL log a warning indicating missing ClassPromotions configuration

#### Scenario: Course with seats available via AdjustmentCourse override is returned
- **WHEN** a student requests available courses during the add/drop phase
- **AND** a class has no matching `ClassPromotions` record for the student's promotion
- **AND** no `SimulationAdjustmentCourse` record exists
- **AND** an `AdjustmentCourse` record exists with `seatCapacity > enrolledCount`
- **THEN** the course-section SHALL appear in the available courses list with status `available`
- **AND** the system SHALL log a warning indicating missing ClassPromotions configuration

#### Scenario: Course with zero capacity across all sources is excluded
- **WHEN** a student requests available courses during the add/drop phase
- **AND** a class has zero seats from `ClassPromotions`, no `SimulationAdjustmentCourse`, and no `AdjustmentCourse` override
- **THEN** the course-section SHALL NOT appear in the available courses list

### Requirement: Real-time seat availability SHALL be displayed per section
The available courses API response SHALL include `available_seats`, `total_seats`, and `enrolled_count` for each course-section. These values SHALL be computed in real-time from current database state (no caching). The existing response fields SHALL remain unchanged (additive only).

#### Scenario: Seat availability fields returned in available course response
- **WHEN** the student requests available courses during the add/drop phase
- **THEN** each course-section in the response SHALL include:
  - `total_seats`: integer, total capacity from the highest-priority source (SimulationAdjustmentCourse > AdjustmentCourse > ClassPromotions)
  - `available_seats`: integer, `total_seats - enrolled_count` (minimum 0)
  - `enrolled_count`: integer, count of ENROLLED bids for this class in this campaign
- **AND** all existing response fields SHALL remain present and unchanged

#### Scenario: Seat counts reflect most recent enrollment state
- **WHEN** Student A enrolls in a section reducing available seats from 2 to 1
- **AND** Student B subsequently requests the available courses list
- **THEN** Student B SHALL see `available_seats: 1` for that section

### Requirement: Students SHALL be able to filter and sort courses by seat availability
The available courses endpoint SHALL accept optional `availability` and `sort_by` query parameters. When omitted, existing default behavior SHALL be preserved.

#### Scenario: Filter to show only courses with available seats
- **WHEN** a student requests available courses with `availability=available`
- **THEN** only course-sections with `available_seats > 0` SHALL be returned
- **AND** full (zero available seats) course-sections SHALL be excluded

#### Scenario: Filter to show all courses including full
- **WHEN** a student requests available courses with `availability=all` or no `availability` parameter
- **THEN** all course-sections SHALL be returned regardless of seat availability (existing behavior)

#### Scenario: Sort courses by available seats descending
- **WHEN** a student requests available courses with `sort_by=available_seats_desc`
- **THEN** course-sections SHALL be ordered by `available_seats` descending, with ties broken by course name ascending

#### Scenario: Default sort order preserved when sort_by omitted
- **WHEN** a student requests available courses without a `sort_by` parameter
- **THEN** existing sort order SHALL be preserved (no behavioral change)

### Requirement: Concurrent enrollments SHALL NOT cause seat over-allocation
The system SHALL use database-level pessimistic locking when checking seat availability and creating enrollment bids. Two students attempting to enroll in the last available seat simultaneously SHALL result in exactly one enrollment and one waitlist entry.

#### Scenario: Two students enroll concurrently for last seat
- **WHEN** a class has exactly 1 available seat
- **AND** Student A and Student B both submit enrollment requests simultaneously
- **THEN** exactly one student SHALL receive ENROLLED status
- **AND** the other student SHALL receive WAITLISTED status with `waitlistRank = 1`
- **AND** the class SHALL show `available_seats: 0` after both requests complete

#### Scenario: Sequential enrollments decrement seats correctly
- **WHEN** a class has 3 available seats
- **AND** Student A enrolls successfully (ENROLLED)
- **AND** Student B then enrolls successfully (ENROLLED)
- **THEN** the class SHALL show `available_seats: 1`

#### Scenario: Enrollment rejected when seats reach zero
- **WHEN** a class has 0 available seats
- **AND** a student submits an enrollment request for that class
- **THEN** the student SHALL receive WAITLISTED status (not ENROLLED)
- **AND** the student SHALL receive a waitlist rank

### Requirement: Direct enrollment SHALL update student enrollment list immediately
Upon successful enrollment (ENROLLED status), the course SHALL appear in the student's enrollment list without requiring manual refresh. The UI SHALL invalidate relevant cached data after enrollment submission.

#### Scenario: Successful enrollment appears in enrollment list
- **WHEN** a student enrolls in a course with available seats
- **AND** the enrollment succeeds with ENROLLED status
- **THEN** the student's enrollment list SHALL immediately include the newly enrolled course
- **AND** the available courses list SHALL show the updated (decremented) seat count

#### Scenario: Waitlisted enrollment appears in waitlist
- **WHEN** a student adds a course with no available seats
- **AND** the bid is created with WAITLISTED status
- **THEN** the student's waitlist SHALL immediately include the course with its waitlist rank
- **AND** the available courses list SHALL show `available_seats: 0` for that section

### Requirement: Student eligibility rules SHALL be enforced before direct enrollment
Direct enrollment SHALL be subject to existing eligibility constraints. The system SHALL prevent enrollment if any constraint is violated, returning a clear error message.

#### Scenario: Enrollment blocked when credit limit exceeded
- **WHEN** a student attempts to enroll in a course
- **AND** enrolling would cause the student's total credits to exceed `maxCreditsToFulfill`
- **THEN** the enrollment SHALL be rejected with an error indicating credit limit exceeded

#### Scenario: Enrollment blocked on timetable conflict
- **WHEN** a student attempts to enroll in a course
- **AND** the class schedule conflicts with another class the student is already enrolled in
- **THEN** the enrollment SHALL be rejected with an error indicating a timetable conflict

#### Scenario: Enrollment allowed when all eligibility rules pass
- **WHEN** a student attempts to enroll in a course with available seats
- **AND** the student's credit total is within limits
- **AND** no timetable conflicts exist
- **AND** the student has not already enrolled in or been waitlisted for the same course
- **THEN** the enrollment SHALL succeed with ENROLLED status
