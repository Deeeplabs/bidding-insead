## ADDED Requirements

### Requirement: Add/Drop dropdown SHALL disable courses the student is already enrolled in

In the Add/Drop & Waitlist phase, the course selection dropdown SHALL preemptively disable courses where the student has an existing enrollment. Enrolled courses SHALL appear with a "Previously Enrolled" label and be visually disabled — students cannot select them from the dropdown.

This is a UI-level enforcement that prevents the error from occurring on save/submit. The backend `validateNoPreviousEnrollment` check remains as a safety net.

#### Scenario: Student sees enrolled course disabled in Add/Drop dropdown
- **GIVEN** student "John Doe" is already enrolled in course "Advanced Group Dynamics" (AA) from a prior campaign
- **AND** the Add/Drop & Waitlist phase is active
- **WHEN** the student opens the "Add Courses" dropdown
- **THEN** "Advanced Group Dynamics (AA)" is displayed but visually disabled
- **AND** the disabled reason shows "Previously Enrolled"
- **AND** clicking on it does NOT add it to the course selection

#### Scenario: Enrolled course blocked even if it has available seats
- **GIVEN** student "Jane Smith" is enrolled in "Finance 101" from a prior campaign
- **AND** "Finance 101" still has available seats in the current Add/Drop phase
- **WHEN** she opens the Add Courses dropdown
- **THEN** all sections of "Finance 101" are disabled with "Previously Enrolled" reason
- **AND** she cannot select any section of "Finance 101"

#### Scenario: Non-enrolled courses remain selectable
- **GIVEN** student "John Doe" is enrolled in "Advanced Group Dynamics" but NOT in "Marketing 201"
- **WHEN** he opens the Add Courses dropdown
- **THEN** "Marketing 201" sections are selectable (not disabled)
- **AND** "Advanced Group Dynamics" sections are disabled

#### Scenario: Enrolled course validation fires as safety net on handleAddCourse
- **GIVEN** a course with `is_enrolled: true` somehow bypasses dropdown disabling
- **WHEN** `handleAddCourse` is called with this course in `use-add-drop-waitlist-form.tsx`
- **THEN** `validateCourseAddition()` returns `isValid: false` with message "Course X has been previously enrolled."
- **AND** a toast error notification is shown
- **AND** the course is NOT added to the form

### Requirement: Bidding round dropdown does NOT block same-course across parallel rounds

In the Bidding phase, the course selection dropdown SHALL NOT disable courses that the student has bid on in other parallel bidding rounds. Students can freely select the same course in BIDDING1 and BIDDING2.

The only courses disabled in the bidding dropdown are:
1. Previously enrolled courses (from prior campaigns): disabled with "Previously Enrolled" reason
2. Time-conflicting courses: disabled with conflict message
3. Already-selected courses (in current form): disabled with "Selected" reason

#### Scenario: Student can bid on same course in parallel bidding rounds
- **GIVEN** student "John Doe" has submitted a bid for "Finance 101" in BIDDING1
- **AND** BIDDING2 is also active
- **WHEN** he opens the Add Courses dropdown in BIDDING2
- **THEN** "Finance 101" sections are NOT disabled
- **AND** he can select "Finance 101" in BIDDING2 as well
