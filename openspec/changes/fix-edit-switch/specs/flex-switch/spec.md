## MODIFIED Requirements

### Requirement: Flex Switch Class Configuration Filtering
The system MUST return courses for the flex switch functionality based on strict filters, ensuring backward compatibility while enriching the payload.

#### Scenario: Valid Core Course Filtered by Programme and Promotion
- **WHEN** a student requests `/v2/api/flex-switch-class-configuration`
- **THEN** only courses matching the student's Programme and Promotion, AND having a type of `Core`, SHALL be returned
- **AND** the payload MUST include the `module` and `term` details for each course configuration
