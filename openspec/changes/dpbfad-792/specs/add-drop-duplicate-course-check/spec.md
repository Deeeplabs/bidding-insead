## ADDED Requirements

### Requirement: Force resolution of duplicate enrollments from parallel bidding rounds (within current module)

When a student has the same course ENROLLED or SELECTED from multiple bidding rounds (e.g., bid round 1 and bid round 2) within the same campaign, the Add/Drop submission SHALL reject the request unless the student's drops resolve all duplicate enrollments down to at most one per course. This validation runs unconditionally on every add/drop submission (including drop-only requests).

**Important**: The duplicate detection is scoped to the current module being submitted to. This means:
- Add/Drop in bid1 only checks for duplicates within bid1's module
- Add/Drop in bid2 only checks for duplicates within bid2's module
- A student can have duplicate enrollments in bid1 AND bid2 simultaneously without conflict

#### Scenario: Student has same course enrolled from bid round 1 and bid round 2, submits add/drop without dropping one
- **GIVEN** student "John Doe" has an ENROLLED bid for class 1501 (course "Finance 101", section EA) from bid round 1 in campaign 10
- **AND** student "John Doe" has an ENROLLED bid for class 1502 (course "Finance 101", section EB) from bid round 2 in campaign 10
- **WHEN** the student submits an Add/Drop request for campaign 10 with enrollment for class 1601 (course "Marketing 201") and no drops
- **THEN** the system rejects the request with a `\DomainException`
- **AND** the error message is: "Duplicate enrollment detected for course Finance 101. You are enrolled in this course from multiple bidding rounds. Please drop one enrollment before submitting."
- **AND** no bids are created or modified

#### Scenario: Student with duplicates in bid1 can submit Add/Drop in bid2 without resolving bid1 duplicates
- **GIVEN** student "John Doe" has ENROLLED bids for course "Finance 101" from bid round 1 (module 1) in campaign 10
- **AND** student "John Doe" also has ENROLLED bids for course "Finance 101" from bid round 1 (creating duplicates within the same module)
- **WHEN** the student submits an Add/Drop request for bid round 2 (module 2) for campaign 10
- **THEN** the system checks for duplicates only within module 2
- **AND** since bid round 2 has no duplicates, the submission is allowed to proceed (duplicates in bid round 1 are independent)

#### Scenario: Student has same course enrolled from two rounds, drops one to resolve
- **GIVEN** student "John Doe" has an ENROLLED bid for class 1501 (course "Finance 101", section EA) from bid round 1 in campaign 10
- **AND** student "John Doe" has an ENROLLED bid for class 1502 (course "Finance 101", section EB) from bid round 2 in campaign 10
- **WHEN** the student submits an Add/Drop request for campaign 10 with:
  - DROP: class 1502 (course "Finance 101", section EB)
- **THEN** the system accepts the submission (duplicate resolved by dropping one)
- **AND** the bid for class 1502 is marked as DROPPED

#### Scenario: Student has same course enrolled from two rounds, drops one and adds another course
- **GIVEN** student "John Doe" has an ENROLLED bid for class 1501 (course "Finance 101", section EA) from bid round 1 in campaign 10
- **AND** student "John Doe" has an ENROLLED bid for class 1502 (course "Finance 101", section EB) from bid round 2 in campaign 10
- **WHEN** the student submits an Add/Drop request for campaign 10 with:
  - DROP: class 1501 (course "Finance 101", section EA)
  - ADD: class 1601 (course "Marketing 201")
- **THEN** the system accepts the submission (duplicate resolved by drop)
- **AND** the bid for class 1501 is marked as DROPPED
- **AND** a new ENROLLED bid is created for class 1601

#### Scenario: Student has multiple duplicate courses from parallel bidding, must resolve all
- **GIVEN** student "John Doe" has ENROLLED bids for course "Finance 101" from bid round 1 AND bid round 2
- **AND** student "John Doe" has ENROLLED bids for course "Strategy 301" from bid round 1 AND bid round 2
- **WHEN** the student submits an Add/Drop request dropping only one Finance 101 enrollment (but not resolving Strategy 301)
- **THEN** the system rejects the request with a `\DomainException` mentioning "Strategy 301"
- **AND** no bids are created or modified

#### Scenario: No duplicate enrollments exist, add/drop proceeds normally
- **GIVEN** student "John Doe" has an ENROLLED bid for class 1501 (course "Finance 101") from bid round 1 in campaign 10
- **AND** student "John Doe" has an ENROLLED bid for class 1601 (course "Marketing 201") from bid round 2 in campaign 10
- **WHEN** the student submits an Add/Drop request for campaign 10
- **THEN** the system accepts the submission (no duplicate courses, different courses in each round)

#### Scenario: Drop-only submission also enforces duplicate resolution
- **GIVEN** student "John Doe" has an ENROLLED bid for course "Finance 101" from bid round 1 AND bid round 2
- **WHEN** the student submits a drop-only Add/Drop request that does NOT drop any Finance 101 enrollment
- **THEN** the system rejects the request because unresolved duplicates exist

### Requirement: Prevent adding a course in Add/Drop 2 that was already added in Add/Drop 1

When a student adds a course during Add/Drop 1 (creating an ENROLLED or WAITLISTED bid), attempting to add the same course in Add/Drop 2 SHALL be rejected regardless of whether a different class section is selected. Both Add/Drop modules share the same Campaign, so `validateNoDuplicateCoursesWithCurrentEnrollment` detects this via a campaign-scoped query.

**Important — module-scoped drop exclusion**: When computing which courses to exclude from the duplicate check (via the `drops` list in the submission), the system SHALL only exclude courses if the student has an actual droppable bid in the **current module** for that class. A class dropped from Add/Drop 2's drop list that belongs to Add/Drop 1 (different module) MUST NOT bypass the duplicate check. Specifically: `validateNoDuplicateCoursesWithCurrentEnrollment` must receive `$moduleId` and use it to verify that any class in `$drops` actually has an ENROLLED/SELECTED bid in the current module before excluding its course from the duplication map.

#### Scenario: Student adds course in Add/Drop 1, tries to add same course (same section) in Add/Drop 2
- **GIVEN** student "Jane Smith" submitted an Add/Drop request in Add/Drop 1 (campaign module sequence 3, moduleId=3) for campaign 10, enrolling in class 1501 (course "Finance 101", section EA)
- **AND** the bid for class 1501 has status ENROLLED, campaign_id=10, campaign_module_id=3
- **WHEN** the student submits an Add/Drop request in Add/Drop 2 (moduleId=5) for campaign 10 with enrollment for class 1501 (same class)
- **THEN** the system rejects the request with a `\DomainException`
- **AND** the error message is: "Already enrolled in Finance 101 (EA)"

#### Scenario: Student adds course in Add/Drop 1, tries to add same course (different section) in Add/Drop 2
- **GIVEN** student "Jane Smith" submitted an Add/Drop request in Add/Drop 1 (moduleId=3) for campaign 10, enrolling in class 1501 (course "Finance 101", section EA)
- **AND** the bid for class 1501 has status ENROLLED in campaign 10
- **WHEN** the student submits an Add/Drop request in Add/Drop 2 (moduleId=5) for campaign 10 with enrollment for class 1502 (course "Finance 101", section EB)
- **THEN** the system rejects the request with a `\DomainException`
- **AND** the error message is: "Already enrolled in course Finance 101 in this campaign. Drop the existing enrollment first before adding a different section."

#### Scenario: Student adds course in Add/Drop 1 (waitlisted), tries to add same course in Add/Drop 2
- **GIVEN** student "Jane Smith" submitted in Add/Drop 1 (moduleId=3) for class 1501 (course "Finance 101") and was WAITLISTED (class full)
- **WHEN** the student submits in Add/Drop 2 (moduleId=5) with enrollment for class 1502 (course "Finance 101", different section)
- **THEN** the system rejects the request with a `\DomainException`
- **AND** the error message is: "Already waitlisted for course Finance 101 in this campaign. Remove from waitlist first before adding a different section."

#### Scenario: Cross-module drop bypass is rejected
- **GIVEN** student "Jane Smith" enrolled in class 1501 (course "Finance 101", section EA) via Add/Drop 1 (moduleId=3), bid has campaign_module_id=3
- **WHEN** the student submits Add/Drop 2 (moduleId=5) with:
  - drops: [class 1501] (which belongs to moduleId=3, not moduleId=5)
  - enrollments: [class 1502 (course "Finance 101", section EB)]
- **THEN** the system rejects the request with a `\DomainException`
- **AND** the error message is: "Already enrolled in course Finance 101 in this campaign. Drop the existing enrollment first before adding a different section."
- **AND** class 1501 is NOT dropped (it belongs to Add/Drop 1, not Add/Drop 2)
- **WHY**: The `drops` list in a moduleId=5 submission MUST NOT be used to bypass duplicate detection for courses enrolled under a different module. The system checks whether the dropped class has an enrolled bid in the current module; if not, the course is NOT excluded from the duplicate check.

#### Scenario: Student adds different courses in Add/Drop 1 and Add/Drop 2
- **GIVEN** student "Jane Smith" enrolled in class 1501 (course "Finance 101") during Add/Drop 1 (moduleId=3)
- **WHEN** the student submits in Add/Drop 2 (moduleId=5) with enrollment for class 1601 (course "Marketing 201")
- **THEN** the system accepts the submission (different courses, no conflict)

#### Scenario: Legitimate drop-then-add within the SAME module still works
- **GIVEN** student "Jane Smith" enrolled in class 1501 (course "Finance 101", section EA) via Add/Drop 1 (moduleId=3), bid has campaign_module_id=3
- **WHEN** the student submits Add/Drop 1 (moduleId=3) again with:
  - drops: [class 1501] (belongs to moduleId=3 — valid current-module drop)
  - enrollments: [class 1502 (course "Finance 101", section EB)]
- **THEN** the system accepts the submission (drop is in the same module, so Finance 101 is correctly excluded from the duplicate check)
- **AND** bid for class 1501 is marked DROPPED
- **AND** new bid for class 1502 is created

### Requirement: Reject duplicate courses within a single Add/Drop submission

When a student submits an Add/Drop request containing multiple items for the same course (different sections) — across enrollment items and waitlist-retain items combined — the system SHALL reject the entire submission with a clear error message.

#### Scenario: Student submits two sections of the same course in one Add/Drop request
- **GIVEN** student "John Doe" is in an open campaign with an active add_drop_waitlist phase
- **WHEN** the student submits an Add/Drop request with enrollments for class 1501 (course "Finance 101", section EA) AND class 1502 (course "Finance 101", section EB)
- **THEN** the system rejects the request with a `\DomainException`
- **AND** the error message is: "Duplicate course detected in submission: Finance 101. Each course can only be enrolled once."
- **AND** no bids are created or modified

#### Scenario: Duplicate course across enrollment and waitlist-retain items
- **GIVEN** student "John Doe" is in an open campaign with an active add_drop_waitlist phase
- **WHEN** the student submits an Add/Drop request with enrollment for class 1501 (course "Finance 101", section EA) AND waitlist-retain for class 1502 (course "Finance 101", section EB)
- **THEN** the system rejects the request with a `\DomainException`
- **AND** the error message is: "Duplicate course detected in submission: Finance 101. Each course can only be enrolled once."

#### Scenario: Distinct courses in submission pass validation
- **GIVEN** student "John Doe" is in an open campaign
- **WHEN** the student submits an Add/Drop request with enrollment for class 1501 (course "Finance 101") AND class 1601 (course "Marketing 201")
- **THEN** the system accepts the submission (passes duplicate course validation)

### Requirement: Enforce capital (bid points) validation during Add/Drop

The system SHALL validate that the student has sufficient capital (bid points) when submitting enrollment items during Add/Drop & Waitlist. If the student's remaining capital plus refunds from drops minus points to spend would result in negative capital, the submission SHALL be rejected.

#### Scenario: Student submits add/drop with insufficient capital
- **GIVEN** student "John Doe" has remaining capital of 50 points
- **AND** the student is not dropping any courses (no refund)
- **WHEN** the student submits an Add/Drop request with enrollment for class 1501 requiring 80 bid points
- **THEN** the system rejects the request with a `\DomainException`
- **AND** the error message includes: "Insufficient bid points. Available: 50, Refund: 0, Required: 80, Deficit: 30"

#### Scenario: Student submits add/drop with capital offset by drops
- **GIVEN** student "John Doe" has remaining capital of 50 points
- **AND** the student is dropping a course that refunds 40 points
- **WHEN** the student submits an Add/Drop request with enrollment requiring 80 bid points
- **THEN** the system accepts the submission (50 + 40 - 80 = 10 remaining, non-negative)

#### Scenario: Student submits add/drop with zero bid points
- **GIVEN** student "John Doe" has remaining capital of 0 points
- **WHEN** the student submits an Add/Drop request with enrollment requiring 0 bid points
- **THEN** the system accepts the submission (0 + 0 - 0 = 0, non-negative)

#### Scenario: Drop-only submission skips capital validation
- **GIVEN** student "John Doe" has remaining capital of -10 points (legacy data)
- **WHEN** the student submits a drop-only Add/Drop request (no enrollments)
- **THEN** the system accepts the submission (capital validation only runs when adding courses)

### Requirement: Drop-then-add same course (section switch) remains valid

The existing drop-then-add pattern SHALL continue to work. When a student drops section EA and adds section EB of the same course in the same request, the duplicate course detection SHALL account for the drop and allow the addition.

#### Scenario: Student drops one section and adds another section of same course
- **GIVEN** student "Jane Smith" has an ENROLLED bid for class 1501 (course "Finance 101", section EA) in campaign 10
- **WHEN** the student submits an Add/Drop request with:
  - DROP: class 1501 (course "Finance 101", section EA)
  - ADD: class 1502 (course "Finance 101", section EB)
- **THEN** the system accepts the submission
- **AND** the bid for class 1501 is marked as DROPPED
- **AND** a new bid is created for class 1502

### Requirement: Cross-campaign duplicate detection remains functional

The existing `validateNoPreviousEnrollment()` SHALL continue to prevent enrollment in courses that were enrolled in other campaigns within the same programme.

#### Scenario: Student enrolled in course via Campaign A, tries to add same course via Campaign B add/drop
- **GIVEN** student "John Doe" has an ENROLLED bid for course "Finance 101" in Campaign A (same programme)
- **AND** Campaign B is running in parallel with an active add_drop_waitlist phase
- **WHEN** the student submits an Add/Drop request for Campaign B with enrollment for a section of "Finance 101"
- **THEN** the system rejects the request with a `\DomainException`
- **AND** the error message is: "Course Finance 101 has been previously enrolled. You cannot enroll in the same course again for this programme."

### Requirement: No impact on existing behavior

#### Scenario: Existing Add/Drop operations continue to work
- **WHEN** a student submits a valid Add/Drop request with no duplicate courses, no unresolved bidding duplicates, and sufficient capital
- **THEN** all existing validation (deadline, eligibility, enrollment status, credit limits, drops) continues to function identically
- **AND** API response shapes remain unchanged
- **AND** no new database migrations are required
