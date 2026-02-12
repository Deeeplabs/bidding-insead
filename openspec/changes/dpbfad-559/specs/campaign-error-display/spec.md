## ADDED Requirements

### Requirement: Campaign save errors are displayed with field-level detail

When a campaign save operation (Save Draft or Save) fails, the system SHALL display a structured error message that identifies the specific fields and issues causing the failure.

#### Scenario: Validation error with single field
- **WHEN** user submits a campaign with an invalid field (e.g., duplicate campaign_name)
- **THEN** system displays a snackbar with message:
  - "Validation failed: campaign_name - This value is already used"
  - Auto-dismisses after 8 seconds
  - Error variant styling

#### Scenario: Validation error with multiple fields
- **WHEN** user submits a campaign with multiple invalid fields
- **THEN** system displays a snackbar message listing all field errors:
  - "Validation failed: campaign_name (already used), start_date (invalid date)"
  - Shows first 3-5 field errors to avoid message truncation
  - Auto-dismisses after 10 seconds for longer messages

#### Scenario: Network error during save
- **WHEN** save operation fails due to network connectivity
- **THEN** system displays error message: "Unable to connect to server. Please check your connection and try again."
- **AND** form data is preserved for retry

#### Scenario: Server error during save
- **WHEN** save operation fails with HTTP 500 error
- **THEN** system displays error message: "A server error occurred. Please try again later."
- **AND** error details are logged for debugging

### Requirement: Error display distinguishes between error types

The system SHALL use different message formats and durations for different categories of errors to help users understand severity and required actions.

#### Scenario: Validation error display
- **WHEN** error code is "VALIDATION_ERROR"
- **THEN** system displays snackbar with error variant
- **AND** uses "Validation failed" prefix
- **AND** shows field-specific errors in compact format
- **AND** auto-dismisses after 8 seconds

#### Scenario: Authorization error display
- **WHEN** error code is "ACCESS_DENIED" or "UNAUTHORIZED"
- **THEN** system displays snackbar with error variant
- **AND** shows message: "You do not have permission to perform this action"
- **AND** does not show field-level details
- **AND** auto-dismisses after 6 seconds

#### Scenario: Generic error display
- **WHEN** error code is not specifically handled
- **THEN** system displays snackbar with error variant
- **AND** shows the error message from the API response
- **AND** includes error code for support reference
- **AND** auto-dismisses after 6 seconds

### Requirement: Error messages are dismissible and preserve form state

The error display SHALL allow users to continue working with the form with all entered data preserved after errors occur.

#### Scenario: Error message auto-dismiss
- **WHEN** snackbar auto-dismisses after timeout
- **THEN** form remains visible with all previously entered data
- **AND** user can immediately correct the errors
- **AND** user can retry the save operation

#### Scenario: Form data preserved after error
- **WHEN** save operation fails
- **THEN** all form field values remain unchanged
- **AND** user can immediately correct the errors
- **AND** user can retry the save operation

### Requirement: Error parsing utility formats messages for snackbar

The system SHALL provide a utility function that parses API error responses and formats them appropriately for snackbar display.

#### Scenario: Parse AxiosError with validation details
- **WHEN** error is an AxiosError with response.data.error.details containing field errors
- **THEN** utility returns:
  ```typescript
  {
    title: "Validation Failed",
    message: "Please correct the following errors:",
    fieldErrors: {
      "campaign_name": ["This value is already used"],
      "start_date": ["This value should be a valid date"]
    },
    code: "VALIDATION_ERROR",
    isValidationError: true
  }
  ```

#### Scenario: Parse AxiosError with generic error
- **WHEN** error is an AxiosError with response.data.error but no details
- **THEN** utility returns:
  ```typescript
  {
    title: "Error",
    message: error.response.data.error.message,
    fieldErrors: {},
    code: error.response.data.error.code,
    isValidationError: false
  }
  ```

#### Scenario: Parse network error
- **WHEN** error is an AxiosError with no response (network failure)
- **THEN** utility returns:
  ```typescript
  {
    title: "Connection Error",
    message: "Unable to connect to server. Please check your connection and try again.",
    fieldErrors: {},
    code: "NETWORK_ERROR",
    isValidationError: false
  }
  ```

#### Scenario: Parse unknown error type
- **WHEN** error is not an AxiosError
- **THEN** utility returns:
  ```typescript
  {
    title: "Unexpected Error",
    message: "An unexpected error occurred. Please try again.",
    fieldErrors: {},
    code: "UNKNOWN_ERROR",
    isValidationError: false
  }
  ```

### Requirement: Error formatting utility creates snackbar-ready messages

The system SHALL provide utility functions that format parsed errors into appropriate snackbar messages.

#### Scenario: Format validation error for snackbar
- **WHEN** formatErrorForSnackbar receives parsed error with fieldErrors
- **THEN** function returns string:
  - Single field: "Validation failed: campaign_name - This value is already used"
  - Multiple fields: "Validation failed: campaign_name (already used), start_date (invalid date)"
- **AND** limits to first 3-5 field errors to avoid truncation

#### Scenario: Format network error for snackbar
- **WHEN** formatErrorForSnackbar receives network error
- **THEN** function returns: "Connection failed. Please check your network and try again."
- **AND** suggests appropriate auto-dismiss duration

#### Scenario: Determine auto-dismiss duration
- **WHEN** getSnackbarDuration receives parsed error
- **THEN** function returns:
  - 8000ms for validation errors (longer for reading)
  - 6000ms for authorization/permission errors
  - 6000ms for generic errors
  - 10000ms for multi-field validation errors

### Requirement: Campaign components use unified error handling

All campaign save operations SHALL use the unified error handling pattern with enhanced snackbar messages for consistent user experience.

#### Scenario: Create campaign save error
- **WHEN** user saves a new campaign and an error occurs
- **THEN** create-campaign.tsx displays enhanced snackbar message
- **AND** form data is preserved

#### Scenario: Edit campaign save error
- **WHEN** user saves an existing campaign and an error occurs
- **THEN** preview-campaign.tsx displays enhanced snackbar message
- **AND** form data is preserved

#### Scenario: Preset campaign save error
- **WHEN** user saves a preset-based campaign and an error occurs
- **THEN** preset-campaign.tsx displays enhanced snackbar message
- **AND** form data is preserved
