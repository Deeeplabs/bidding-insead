## Campaign Error Display - Structured Error Messages for Campaign Save Operations

### Problem
**Jira**: https://insead.atlassian.net/browse/DPBFAD-559

When campaign save operations fail in the bidding-admin application, users receive generic error messages like "Failed to save draft campaign" without detailed information about what went wrong. This makes it difficult for users to:
1. Understand the root cause of the error
2. Take corrective action based on specific validation failures
3. Identify which form fields need attention

### Solution

Implement a structured error display system that:
1. Parses API error responses into a consistent format
2. Displays user-friendly error messages in enhanced snackbars
3. Shows field-specific validation errors when available
4. Handles various error types (validation, network, server errors)
5. Uses appropriate auto-dismiss timing based on error severity

### Changes

| File | Change |
|------|--------|
| `src/utils/error-parser.ts` | New utility for parsing API errors into consistent `ParsedError` format |
| `src/utils/__tests__/error-parser.test.ts` | Unit tests for error parser covering all error scenarios |
| `src/utils/index.ts` | Export file for utility functions |
| `src/components/campaign/create-campaign.tsx` | Integration of enhanced snackbar error messages for draft save operations |
| `src/components/campaign/preview-campaign.tsx` | Integration of enhanced snackbar error messages for submit and draft save operations |
| `src/components/campaign/preset-campaign.tsx` | Integration of enhanced snackbar error messages for draft save operations |

### Technical Details

#### Error Parser Utility
- Handles AxiosError responses with validation details
- Extracts field-level errors from API response
- Provides user-friendly field name mapping
- Handles network errors and unknown error types

#### Enhanced Snackbar Messages
- Material-UI Snackbar-based error display
- Displays error title and detailed message
- Shows field errors in a readable, comma-separated format
- Supports auto-dismiss with configurable timing
- Non-blocking - allows continued form interaction

#### Integration Pattern
Each campaign component was updated to:
1. Import `parseApiError`, `formatErrorForSnackbar`, and `getSnackbarDuration` functions
2. Update catch blocks to use enhanced snackbar messages
3. Remove error modal state management (using snackbar approach)
4. Add appropriate auto-dismiss timing based on error type

### Testing

- **Unit Tests**: Error parser utility has comprehensive test coverage
- **Manual Testing Required**:
  - Test duplicate campaign name error
  - Test invalid date validation error
  - Test network error handling
  - Verify form data preservation after error
  - Smoke test successful operations still work

### Impact

- No API changes required
- No database migrations needed
- Backward compatible with existing error responses
- Improved user experience for error handling
- Non-breaking change - existing functionality preserved

### Screenshots

**Before**: Generic snackbar error message
```
Failed to save draft campaign
```

**After**: Enhanced snackbar with detailed error information
```
┌─────────────────────────────────────────────────────────┐
│  ⚠️ Validation failed: campaign_name (already used),    │
│  start_date (must be a future date)                    │
│                                                     [×] │
└─────────────────────────────────────────────────────────┘
```
