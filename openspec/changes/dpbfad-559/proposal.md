## Why

When saving a campaign (Save Draft or Save), errors from the API are displayed as generic messages like "Failed to save draft campaign" without showing the actual validation errors or field-specific issues. This prevents business users (PMs) from understanding what went wrong and how to fix it, potentially causing loss of work and frustration.

## What Changes

- Frontend will parse and display detailed error messages from API responses, including:
  - Field-specific validation errors (e.g., "campaign_name: This value is already used")
  - Error codes for categorization (VALIDATION_ERROR, BAD_REQUEST, etc.)
  - Distinguish between blocking errors and warnings
  
- Error display will be enhanced to:
  - Show errors in enhanced snackbar messages with detailed information
  - Display field-specific validation errors in readable format
  - Use appropriate auto-dismiss timing based on error type
  - Preserve user input on error (no data loss)
  - Allow users to continue interacting with the form while errors are displayed

## Capabilities

### New Capabilities
- `campaign-error-display`: Clear, structured error message display for campaign save operations with field-level error identification and warning vs blocking error distinction.

### Modified Capabilities
- None (this is a new UI enhancement, not changing existing spec-level behavior)

## Impact

**Frontend (bidding-admin):**
- [`create-campaign.tsx`](bidding-admin/src/components/campaign/create-campaign.tsx) - Enhance error handling in `handleDraftSave()`
- [`preview-campaign.tsx`](bidding-admin/src/components/campaign/preview-campaign.tsx) - Enhance error handling in `handleSaveDraft()` and `handleSave()`
- [`preset-campaign.tsx`](bidding-admin/src/components/campaign/preset-campaign.tsx) - Enhance error handling in `handleDraftSave()`
- New utility functions for structured error parsing and snackbar formatting in `bidding-admin/src/src/utils/error-parser.ts`

**Backend (bidding-api):**
- Already has structured error responses via [`ExceptionSubscriber.php`](bidding-api/src/EventSubscriber/ExceptionSubscriber.php)
- Already returns [`ApiResponse`](bidding-api/src/Controller/Api/ApiResponse.php) with [`ApiError`](bidding-api/src/Controller/Api/ApiError.php) containing `code`, `message`, `details`, and `field`
- No backend changes required - existing error structure is sufficient

**API Error Response Structure (already exists):**
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
    }
  }
}
```

**Affected code patterns:**
- Current pattern: `enqueueSnackbar('Failed to save draft campaign', { variant: 'error' })`
- New pattern: Parse `error.response.data.error.details` and display structured errors in enhanced snackbar messages with field-specific information and appropriate auto-dismiss timing
