## NEW Requirements

### Requirement: Credit and Bidding Capital input validation
The system SHALL validate all Credit and Bidding Capital input fields to ensure only valid numeric values are accepted, with proper error messaging and consistent enforcement across frontend and backend.

#### Scenario: Reject non-numeric characters
- **WHEN** a user enters non-numeric characters (letters, symbols, etc.) in any Credit or Bidding Capital field
- **THEN** the input SHALL be rejected
- **THEN** a clear error message SHALL be displayed: "Only numeric values are allowed"

#### Scenario: Reject negative values
- **WHEN** a user enters a negative value in any Credit or Bidding Capital field
- **THEN** the input SHALL be rejected
- **THEN** a clear error message SHALL be displayed: "Negative values are not allowed"

#### Scenario: Reject values with more than one decimal place
- **WHEN** a user enters a value with more than one decimal place (e.g., "10.555", "5.123") in any Credit or Bidding Capital field
- **THEN** the input SHALL be rejected
- **THEN** a clear error message SHALL be displayed: "Only one decimal place is allowed"

#### Scenario: Accept valid numeric values
- **WHEN** a user enters a valid positive number with zero or one decimal place (e.g., "10", "10.5", "100.0")
- **THEN** the input SHALL be accepted
- **THEN** no error message SHALL be displayed

#### Scenario: Frontend real-time validation
- **GIVEN** a user is typing in a Credit or Bidding Capital field
- **WHEN** invalid input is entered
- **THEN** validation SHALL occur in real-time
- **THEN** error feedback SHALL appear immediately
- **WHEN** the user corrects the input to be valid
- **THEN** the error message SHALL disappear

#### Scenario: Backend API validation
- **GIVEN** a request is submitted to any API endpoint with Credit or Bidding Capital parameters
- **WHEN** the request contains invalid numeric values
- **THEN** the API SHALL return a 400 Bad Request response
- **THEN** the response SHALL include specific validation error messages for each invalid field
- **WHEN** all values are valid
- **THEN** the request SHALL proceed normally

#### Scenario: Campaign configuration validation
- **GIVEN** an admin is configuring campaign settings in the admin dashboard
- **WHEN** entering Credit limits (minCreditsToFulfill, maxCreditsToFulfill)
- **WHEN** entering Bidding Capital limits (minCapitalGranted, maxCapitalGranted)
- **THEN** all the above validation rules SHALL apply
- **THEN** the form SHALL not be submittable until all values are valid

#### Scenario: Module configuration validation
- **GIVEN** an admin is configuring module-specific settings
- **WHEN** entering Credit or Capital ranges for modules
- **WHEN** entering single value limits
- **THEN** all the above validation rules SHALL apply to both min/max and single value inputs

#### Scenario: Student bidding form validation
- **GIVEN** a student is entering bid amounts in the student portal
- **WHEN** allocating bidding capital to courses
- **THEN** validation is NOT required for capital/credit fields (these are read-only display values)
- **THEN** students only input bid points for individual courses, not total capital amounts

#### Scenario: Consistent validation across all forms
- **GIVEN** any form in the system accepts Credit or Bidding Capital input
- **WHEN** validation is performed
- **THEN** the same validation rules and error messages SHALL be applied consistently
- **THEN** both frontend and backend SHALL enforce identical constraints

#### Scenario: Edge case handling
- **WHEN** a user enters "0" as a value
- **THEN** it SHALL be accepted as valid (zero is allowed)
- **WHEN** a user enters "0.0" as a value
- **THEN** it SHALL be accepted as valid
- **WHEN** a user enters an empty value
- **THEN** it SHALL be handled according to field requirements (required vs optional)
- **WHEN** a user enters only a decimal point "."
- **THEN** it SHALL be rejected as invalid

#### Scenario: International number format handling
- **GIVEN** the system is configured for international use
- **WHEN** users enter numbers with different formatting preferences
- **THEN** the system SHALL standardize on decimal point (.) as the decimal separator
- **THEN** comma separators for thousands SHALL be handled or rejected consistently
