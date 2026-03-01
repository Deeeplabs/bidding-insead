## CHANGED Requirements

### Requirement: Simulation phase displays accurate course and section counts

The system SHALL display accurate `total_courses` and `total_sections` counts in the Simulation phase, including when no bids exist yet.

#### Scenario: Course count shows available courses when no bids exist
- **GIVEN** a campaign has been created with courses configured
- **AND** no students have submitted bids yet (`$courseIds` is empty)
- **WHEN** the Programme Manager views the Simulation phase statistics
- **THEN** `total_courses` SHALL reflect the number of available courses from campaign configuration
- **AND** `total_sections` SHALL reflect the number of class sections matching the campaign promotion
- **AND** the available courses SHALL be displayed in the course list

#### Scenario: Course count is calculated before empty-bids check
- **GIVEN** the `getCoursesWithEnrollmentData()` method is called
- **WHEN** `$courseIds` from bids is empty
- **THEN** `totalCourses` and `totalSections` SHALL already be calculated from `$availableCourses`
- **AND** the method SHALL NOT return early with 0 if available courses exist

#### Scenario: Search filter recalculates totals correctly
- **GIVEN** the Programme Manager applies a search filter on courses
- **WHEN** the search filter reduces the available courses
- **THEN** `totalCourses` SHALL be recalculated from the filtered course set
- **AND** `totalSections` SHALL be recalculated from the filtered course set
- **AND** pagination SHALL use the recalculated `totalCourses`

#### Scenario: Empty return includes total_sections
- **GIVEN** both `$courseIds` and `$availableCourses` are empty
- **WHEN** the method returns an empty response
- **THEN** the response SHALL include `total_sections: 0` for consistency

### Requirement: Simulation phase displays same student count as Bidding Round

The Simulation phase SHALL use `CampaignStudentEligibilityService::getEligibleStudents()` for the `total_students` statistic.

#### Scenario: Student count uses eligible students
- **GIVEN** a campaign is in the Simulation phase
- **WHEN** the Programme Manager views Simulation statistics
- **THEN** `total_students` SHALL equal the eligible student count from `getEligibleStudents()` using the simulation phase's config
