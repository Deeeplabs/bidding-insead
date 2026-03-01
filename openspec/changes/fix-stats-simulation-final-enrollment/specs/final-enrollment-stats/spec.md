## CHANGED Requirements

### Requirement: Final Enrollment phase displays same student count as Bidding Round

The system SHALL display the same `total_students` count in all campaign phases. This count represents eligible students for the campaign, NOT students who have performed a specific action.

#### Scenario: Student count matches Bidding Round
- **GIVEN** a campaign has progressed to the Final Enrollment phase
- **WHEN** the Programme Manager views Final Enrollment statistics
- **THEN** the `total_students` SHALL equal the count from `CampaignStudentEligibilityService::getEligibleStudents()` using the bidding round config
- **AND** this count SHALL be identical to the Bidding Round's `total_students`

#### Scenario: Student count is independent of bid activity
- **GIVEN** a campaign has 100 eligible students
- **AND** only 80 students have submitted bids
- **AND** only 60 students have ENROLLED/SELECTED status
- **WHEN** viewing Final Enrollment statistics
- **THEN** `total_students` SHALL be 100 (all eligible students, not 60 enrolled)

#### Scenario: Student count is independent of add/drop activity
- **GIVEN** a campaign has eligible students
- **AND** some students have performed add/drop operations
- **WHEN** viewing Final Enrollment statistics
- **THEN** `total_students` SHALL remain the same as in the Bidding Round phase

### Requirement: Student course queries filter by moduleType

The system SHALL only return bids from the Final Enrollment phase when querying student courses.

#### Scenario: Student courses show only final enrollment bids
- **GIVEN** a student has bids in bidding_round, add_drop, and final_enrollment phases
- **WHEN** viewing the student's courses in Final Enrollment
- **THEN** only bids with `moduleType = 'final_enrollment'` SHALL be returned

#### Scenario: Student enrolled courses show only final enrollment enrollments
- **GIVEN** a student has enrolled courses from multiple phases
- **WHEN** viewing the student's enrolled courses in Final Enrollment
- **THEN** only enrollments with `moduleType = 'final_enrollment'` SHALL be returned
