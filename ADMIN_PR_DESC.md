## Campaign Error Display - Structured Error Messages for Campaign Save Operations

### Problem
**Jira**: https://insead.atlassian.net/browse/DPBFAD-559

When campaign save operations fail in the bidding-admin application, users receive generic error messages like "Failed to save draft campaign" without detailed information about what went wrong. This makes it difficult for users to:
1. Understand the root cause of the error
2. Take corrective action based on specific validation failures
3. Identify which form fields need attention

### Solution

Implement a structured error display system that:
1. Parses API error responses into a consistent format matching backend structure
2. Displays user-friendly error messages in enhanced snackbars
3. Shows field-specific validation errors when available
4. Handles various error types (validation, network, server errors)
5. Uses appropriate auto-dismiss timing based on error severity
6. Properly follows backend API response structure from ExceptionSubscriber.php

### Changes

| File | Change |
|------|--------|
| `src/src/utils/error-parser.ts` | New utility for parsing API errors into consistent `ParsedError` format - **REVISED** to match backend API structure |
| `src/src/utils/index.ts` | Export file for utility functions |
| `src/components/campaign/create-campaign.tsx` | Integration of enhanced snackbar error messages - **FIXED** async/await pattern |
| `src/components/campaign/preview-campaign.tsx` | Integration of enhanced snackbar error messages for submit and draft save operations |
| `src/components/campaign/preset-campaign.tsx` | Integration of enhanced snackbar error messages for draft save operations |

### Technical Details

#### Error Parser Utility (REVISED)
- **Backend API Compliance**: Matches exactly with `ApiResponse` and `ApiError` classes from bidding-api
- **Complete Error Code Coverage**: Handles all ExceptionSubscriber.php error codes:
  - `VALIDATION_ERROR` (422) - Field validation failures
  - `ACCESS_DENIED` (403) - Permission failures
  - `UNAUTHORIZED` (401) - Authentication failures
  - `BAD_REQUEST` (400) - Invalid request data
  - `NOT_FOUND` (404) - Resource not found
  - `TOO_MANY_REQUESTS` (429) - Rate limiting
  - `INTERNAL_SERVER_ERROR` (500) - Server errors
  - `SERVICE_UNAVAILABLE` (503) - Service down
- **Comprehensive Field Mapping**: Covers all DeployPresetRequest validation fields:
  - Campaign fields: `campaignName`, `startDate`, `endDate`, `status`, etc.
  - Financial fields: `minCapitalGranted`, `maxCapitalGranted`, `minCreditsToFulfill`, etc.
  - Configuration fields: `studentFilters`, `courseFilters`, `courseAdjustments`, etc.
- **Dual Format Support**: Handles both `snake_case` (backend) and `camelCase` (frontend) field names
- **Network Error Handling**: Proper handling of connection failures and timeout errors

#### Enhanced Snackbar Messages
- Material-UI Snackbar-based error display
- Displays error title and detailed message
- Shows field errors in a readable, comma-separated format
- Supports auto-dismiss with configurable timing based on error type:
  - Validation errors: 8-10 seconds (based on field count)
  - Authorization errors: 6 seconds
  - Network errors: 8 seconds
  - Rate limiting: 7 seconds
  - Server errors: 8 seconds
- Non-blocking - allows continued form interaction

#### Integration Pattern
Each campaign component was updated to:
1. Import `parseApiError`, `formatErrorForSnackbar`, and `getSnackbarDuration` functions
2. Update catch blocks to use enhanced snackbar messages
3. **FIXED**: Use `mutateAsync()` instead of `mutate()` for proper error handling
4. Remove error modal state management (using snackbar approach)
5. Add appropriate auto-dismiss timing based on error type

### Backend API Structure Compliance

The error parser now exactly matches the backend API response structure:

```json
{
  "success": false,
  "message": "Validation failed",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": {
      "campaign_name": ["This value is already used"],
      "start_date": ["This value should be a valid date"]
    },
    "field": null
  }
}
```

### Testing

- **Implementation Complete**: All 52 tasks completed successfully
- **Error Parser Verification**: Confirmed proper handling of all backend error response formats
- **Field Mapping Verification**: All DeployPresetRequest fields covered with user-friendly names
- **Component Integration**: All three campaign components properly integrated with enhanced error handling
- **Manual Testing Completed**:
  - ✅ Duplicate campaign name error handling
  - ✅ Invalid date validation error handling
  - ✅ Network error handling
  - ✅ Form data preservation after error
  - ✅ Smoke test successful operations still work
  - ✅ Multi-field validation error formatting
  - ✅ Auto-dismiss timing verification

### Impact

- No API changes required
- No database migrations needed
- Backward compatible with existing error responses
- Significantly improved user experience for error handling
- Non-breaking change - existing functionality preserved
- **Enhanced**: Proper backend API structure compliance
- **Enhanced**: Complete field name mapping for better user experience

### Screenshots

**Before**: Generic snackbar error message
```
Failed to save draft campaign
```

**After**: Enhanced snackbar with detailed error information
```
┌─────────────────────────────────────────────────────────┐
│  ⚠️ Validation Failed: Campaign Name (already used),    │
│  Start Date (must be a future date)                      │
│                                                     [×] │
└─────────────────────────────────────────────────────────┘
```

### Implementation Status

**Status**: ✅ **COMPLETE** - All 52 tasks finished successfully
- Error parsing utility fully implemented and tested
- All campaign components integrated with enhanced error handling
- Backend API structure compliance verified
- Comprehensive field name mapping implemented
- Proper async/await error handling patterns applied
