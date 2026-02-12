## Context

Campaign save operations (Save Draft / Save) in the admin dashboard currently display generic error messages like "Failed to save draft campaign" without showing the actual validation errors returned by the API. The backend already returns structured error responses with field-level details, but the frontend doesn't parse or display them.

**Current State:**
- Backend returns structured errors via [`ApiResponse`](bidding-api/src/Controller/Api/ApiResponse.php) with [`ApiError`](bidding-api/src/Controller/Api/ApiError.php) containing `code`, `message`, `details`, and `field`
- Frontend catches errors but only logs to console and shows generic snackbar message
- Users cannot identify which fields need correction

**Affected Components:**
- [`create-campaign.tsx`](bidding-admin/src/components/campaign/create-campaign.tsx) - `handleDraftSave()` at line 191-194
- [`preview-campaign.tsx`](bidding-admin/src/components/campaign/preview-campaign.tsx) - `handleSaveDraft()` at line 264-267, `handleSave()` at line 202-205
- [`preset-campaign.tsx`](bidding-admin/src/components/campaign/preset-campaign.tsx) - `handleDraftSave()` at line 224-226

## Goals / Non-Goals

**Goals:**
- Parse and display structured error messages from API responses
- Show field-specific validation errors in a readable format
- Distinguish between different error types (validation, network, server)
- Preserve user input on error (no data loss)
- Consistent error handling across all campaign save operations

**Non-Goals:**
- Changing backend error response structure (already sufficient)
- Adding new validation rules
- Implementing auto-save functionality
- Handling errors for non-campaign pages

## Decisions

### 1. Error Parsing Utility

**Decision:** Create a reusable utility function to extract and format API errors.

**Rationale:** Multiple components need the same error parsing logic. A utility function ensures consistency and DRY principles.

**Location:** [`bidding-admin/src/src/utils/error-parser.ts`](bidding-admin/src/src/utils/error-parser.ts) (new file)

**Interface:**
```typescript
interface ParsedError {
  title: string;
  message: string;
  fieldErrors: Record<string, string[]>;
  code: string;
  isValidationError: boolean;
}

function parseApiError(error: unknown): ParsedError;
```

**Alternatives Considered:**
- Inline parsing in each component → rejected: code duplication
- Custom React hook → rejected: parsing is not React-specific, utility is simpler
- Axios interceptor → rejected: would affect all API calls globally, may have unintended side effects

### 2. Error Display Pattern

**Decision:** Use enhanced snackbar messages for error display with detailed field information.

**Rationale:** 
- Snackbar is already integrated and familiar to users
- No additional UI components needed
- Simpler implementation and maintenance
- Can show multiple field errors in a single message
- Auto-dismiss after reasonable timeout
- Non-blocking - allows users to continue interacting with the form

**UX Flow:**
1. User clicks Save/Save Draft
2. On error: parse API error and show detailed snackbar message
3. Snackbar displays field errors in readable format
4. Form data is preserved for retry
5. User can fix issues and retry without losing data

**Message Format:**
- Single field: "Validation failed: campaign_name - This value is already used"
- Multiple fields: "Validation failed: campaign_name (already used), start_date (invalid date)"
- Network error: "Connection failed. Please check your network and try again."
- Server error: "Server error occurred. Please try again later."

**Snackbar Behavior:**
- Auto-dismiss duration: 8-10 seconds for validation errors (longer for complex messages)
- Error variant (red styling) for all error types
- Manual dismiss option available
- No action buttons needed (informational only)

**Alternatives Considered:**
- Modal dialog → rejected: blocks interaction, overkill for validation errors
- Inline alert below form → rejected: may not be visible on long forms
- Highlight invalid fields → rejected: requires mapping API field names to form fields
- Toast notifications → rejected: snackbar is already integrated in the app

### 3. Integration Pattern

**Decision:** Update each campaign component's error handling to use enhanced snackbar messages.

**Pattern:**
```typescript
// Before
catch (error) {
  enqueueSnackbar('Failed to save draft campaign', { variant: 'error' });
  console.log(error);
}

// After
catch (error) {
  const parsed = parseApiError(error);
  const message = parsed.isValidationError 
    ? `${parsed.title}: ${Object.entries(parsed.fieldErrors).map(([field, errors]) => 
        `${field} (${errors.join(', ')})`
      ).join(', ')}`
    : parsed.message;
  enqueueSnackbar(message, { variant: 'error', autoHideDuration: 8000 });
}
```

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| API field names may not match user-friendly labels | Create a field name mapping/translation function in error parser |
| Long error messages may exceed snackbar space | Limit field errors to first 3-5, truncate long messages |
| Network errors have different structure | Handle both AxiosError and generic Error types in parseApiError |
| Existing tests may expect old error format | Update tests to verify new error handling |
| Snackbar auto-dismiss may be too fast | Use longer duration (8-10 seconds) for validation errors |

## Migration Plan

1. **Phase 1: Create utilities** (no breaking changes)
   - Add `error-parser.ts` utility with field name mapping
   - Add helper functions for formatting snackbar messages

2. **Phase 2: Update components** (incremental rollout)
   - Update `create-campaign.tsx` error handling first
   - Test thoroughly with various error scenarios
   - Update `preview-campaign.tsx`
   - Update `preset-campaign.tsx`

3. **Phase 3: Polish**
   - Refine field name translations for better UX
   - Add unit tests for error parser
   - Update documentation

**Rollback:** Each component update is independent. Can revert individual error handling changes if issues arise.

## Open Questions

1. Should we add different snackbar variants for different error types?
2. Should we include error codes in snackbar messages for support purposes?
3. Should we add a "Copy error details" action to snackbar?
