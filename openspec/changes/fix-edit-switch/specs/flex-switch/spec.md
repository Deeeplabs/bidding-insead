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
