## ADDED Requirements

### Requirement: Student seat display SHALL be static across Bidding Round and Add/Drop
During both the Bidding Round and Add/Drop phases, students SHALL see only the total configured seat capacity for each course-section. The seat value displayed SHALL NOT change based on enrollment activity. Students SHALL NOT see remaining seats, enrolled counts, or any real-time seat consumption data.

#### Scenario: Student sees static total seats during Bidding Round
- **WHEN** a student views available courses during the Bidding Round phase
- **THEN** each course-section SHALL display only `total_seats` as a static number
- **AND** no `available_seats` or `enrolled_count` SHALL be rendered in the UI
- **AND** the seat value SHALL NOT change as other students submit bids

#### Scenario: Student sees static total seats during Add/Drop
- **WHEN** a student views available courses during the Add/Drop phase
- **THEN** each course-section SHALL display only `total_seats` as a static number
- **AND** no `available_seats` or `enrolled_count` SHALL be rendered in the UI
- **AND** the seat value SHALL NOT change as other students enroll or drop courses

#### Scenario: Seat chip displays neutral styling regardless of capacity state
- **WHEN** a course-section has `total_seats` configured
- **THEN** the seat chip SHALL display `{total_seats} seats` with neutral styling
- **AND** the chip SHALL NOT use green/red coloring based on availability
- **AND** the chip SHALL NOT indicate whether seats are remaining or full

### Requirement: PM dashboard SHALL display seat format based on course sharing
The PM's dashboard course list SHALL display seat capacity in one of two formats depending on whether the course is shared across multiple programme-promotions. The format `XX (XX)` SHALL only appear when the course is genuinely split between programme-promotions.

#### Scenario: Non-shared course displays single seat value
- **WHEN** a PM views the bidding round course list
- **AND** a course-section is assigned to only one programme-promotion
- **THEN** the "Seat Available" column SHALL display only `total_seats` (e.g., `48`)
- **AND** the value SHALL NOT include parenthetical notation

#### Scenario: Shared course displays split seat format
- **WHEN** a PM views the bidding round course list
- **AND** a course-section is shared/split across multiple programme-promotions
- **THEN** the "Seat Available" column SHALL display `total_seats (total_all_seats)` (e.g., `3 (42)`)
- **AND** `total_seats` represents the promotion-specific allocated seats
- **AND** `total_all_seats` represents the total capacity across all sharing promotions

#### Scenario: Split format determined by class promotion count, not value difference
- **WHEN** a course has `total_seats = 18` and `total_all_seats = 48`
- **AND** the course has `ClassPromotions` records for multiple distinct promotions
- **THEN** the PM dashboard SHALL display `18 (48)`

#### Scenario: Non-shared course with adjusted seats does not show split format
- **WHEN** a course has `total_seats = 40` and `total_all_seats = 48`
- **AND** the course has `ClassPromotions` records for only one programme-promotion
- **AND** the difference is due to an `AdjustmentCourse` seat override
- **THEN** the PM dashboard SHALL display only `40` (not `40 (48)`)

### Requirement: Backend SHALL enforce real-time enrollment validation
The system SHALL validate seat capacity in real-time at the point of enrollment. Students SHALL be prevented from enrolling when the course is full. The validation SHALL be invisible to the student until they attempt to add a full course.

#### Scenario: Enrollment succeeds when seats are available
- **WHEN** a student submits an enrollment request for a course-section during Add/Drop
- **AND** the current enrolled count is less than the total seat capacity
- **AND** the student meets credit limits, has no timetable conflicts, and passes eligibility rules
- **THEN** the enrollment SHALL succeed with ENROLLED status
- **AND** the course SHALL appear in the student's enrollment list

#### Scenario: Enrollment blocked when course is full
- **WHEN** a student submits an enrollment request for a course-section during Add/Drop
- **AND** the current enrolled count equals or exceeds total seat capacity
- **THEN** the enrollment SHALL be rejected
- **AND** the system SHALL return a clear error message (e.g., "Course is full")
- **AND** the student SHALL NOT be enrolled

#### Scenario: Concurrent enrollments SHALL NOT cause over-allocation
- **WHEN** a class has exactly 1 available seat
- **AND** Student A and Student B both submit enrollment requests simultaneously
- **THEN** exactly one student SHALL receive ENROLLED status
- **AND** the other student SHALL be rejected with "Course is full" or receive WAITLISTED status
- **AND** total enrolled count SHALL NOT exceed total seat capacity

#### Scenario: Enrollment blocked when credit limit exceeded
- **WHEN** a student attempts to enroll in a course
- **AND** enrolling would cause the student's total credits to exceed `maxCreditsToFulfill`
- **THEN** the enrollment SHALL be rejected with an error indicating credit limit exceeded

#### Scenario: Enrollment blocked on timetable conflict
- **WHEN** a student attempts to enroll in a course
- **AND** the class schedule conflicts with another class the student is already enrolled in
- **THEN** the enrollment SHALL be rejected with an error indicating a timetable conflict

### Requirement: No UI element SHALL reveal real-time seat availability to students
The student-facing UI SHALL NOT contain any visual indication of remaining seats, seat consumption, or real-time availability. This applies to all views in both Bidding Round and Add/Drop phases.

#### Scenario: No remaining seat count shown in course selector
- **WHEN** a student opens the course selector dropdown
- **THEN** no course option SHALL display remaining or available seat counts
- **AND** no course option SHALL display enrolled counts

#### Scenario: No availability-based visual differentiation
- **WHEN** a student views available courses
- **THEN** courses SHALL NOT be color-coded or visually differentiated based on seat availability
- **AND** there SHALL be no "full" or "available" badge derived from real-time seat data

#### Scenario: API response fields preserved for backward compatibility
- **WHEN** the API returns available course data to the student frontend
- **THEN** the response MAY still include `available_seats`, `total_seats`, and `enrolled_count` fields
- **AND** the student UI SHALL NOT render `available_seats` or `enrolled_count` to the user
- **AND** only `total_seats` SHALL be used for display purposes
