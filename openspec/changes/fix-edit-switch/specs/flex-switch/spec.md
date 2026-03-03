## MODIFIED Requirements

### Requirement: Flex Switch Class Configuration Filtering
The system MUST return courses for the flex switch functionality based on strict filters, ensuring backward compatibility while enriching the payload.

#### Scenario: Valid Core Course Filtered by Programme and Promotion
- **WHEN** a student requests `/v2/api/flex-switch-class-configuration`
- **THEN** only courses matching the student's Programme and Promotion, AND having a type of `Core`, SHALL be returned
- **AND** the payload MUST include the `module` and `term` details for each course configuration

### Requirement: Enhanced Search Functionality
The system MUST support searching by module name and campus (home campus) in addition to existing course name, course ID, and section.

#### Scenario: Search by Module Name
- **WHEN** a student provides `search=module` parameter
- **THEN** the system SHALL filter courses where the module name contains the search term (e.g., "Module 1", "Module 2")

#### Scenario: Search by Campus Name or Code
- **WHEN** a student provides `search=Singapore` or `search=SGP` parameter
- **THEN** the system SHALL filter courses where the campus name OR campus code matches the search term

#### Scenario: Combined Search
- **WHEN** a student provides `search=Module 1 Singapore`
- **THEN** the system SHALL return courses matching either the module OR the campus criteria

### Requirement: Flex Switch Notifications
The system MUST dispatch appropriately formatted notifications when flex switch requests are submitted or processed.

#### Scenario: Submitting a request
- **WHEN** a student submits a flex switch request
- **THEN** they MUST immediately receive a notification confirming the submission, with `announcement_title` and `announcement_body` placeholders resolved.
- **AND** all Programme Managers associated with the student's programme MUST receive a bulk notification alerting them to the new request.

#### Scenario: Processing an approval
- **WHEN** a PM or admin processes a flex switch approval request
- **THEN** the student MUST receive a notification confirming the approval or rejection, with the placeholders appropriately resolved.
