## 1. Error Parsing Utility

- [x] 1.1 Create `bidding-admin/src/src/utils/error-parser.ts` with `ParsedError` interface and `parseApiError()` function
- [x] 1.2 Add `formatErrorForSnackbar()` function to format errors for snackbar display
- [x] 1.3 Add `getSnackbarDuration()` function to determine appropriate auto-dismiss timing
- [x] 1.4 Add field name mapping for user-friendly error messages
- [x] 1.5 Export error parser functions from `bidding-admin/src/src/utils/index.ts`

## 2. Error Display Integration

- [x] 2.1 Update error handling pattern to use enhanced snackbar messages
- [x] 2.2 Add auto-dismiss duration logic based on error type
- [x] 2.3 Test snackbar message formatting for different error scenarios

## 3. Update Create Campaign Component

- [x] 3.1 Import `parseApiError`, `formatErrorForSnackbar`, and `getSnackbarDuration` in `bidding-admin/src/components/campaign/create-campaign.tsx`
- [x] 3.2 Update `handleDraftSave()` catch block to use enhanced snackbar messages
- [x] 3.3 Remove error modal state and components (using snackbar approach instead)
- [x] 3.4 Test error handling with various validation error scenarios
- [ ] 3.5 Manual test: Create campaign with duplicate name → verify snackbar shows field error

## 4. Update Preview Campaign Component

- [x] 4.1 Import `parseApiError`, `formatErrorForSnackbar`, and `getSnackbarDuration` in `bidding-admin/src/components/campaign/preview-campaign.tsx`
- [x] 4.2 Update `handleSaveDraft()` catch block to use enhanced snackbar messages
- [x] 4.3 Update `handleSubmitCampaign()` catch block to use enhanced snackbar messages
- [x] 4.4 Remove error modal state and components (using snackbar approach instead)
- [ ] 4.5 Manual test: Edit campaign with invalid data → verify snackbar shows field errors

## 5. Update Preset Campaign Component

- [x] 5.1 Import `parseApiError`, `formatErrorForSnackbar`, and `getSnackbarDuration` in `bidding-admin/src/components/campaign/preset-campaign.tsx`
- [x] 5.2 Update `handleDraftSave()` catch block to use enhanced snackbar messages
- [x] 5.3 Remove error modal state and components (using snackbar approach instead)
- [ ] 5.4 Manual test: Create preset campaign with validation errors → verify snackbar display

## 6. Testing and Verification

- [ ] 6.1 Manual test: Create campaign with duplicate name → verify snackbar shows field error
- [ ] 6.2 Manual test: Create campaign with invalid date → verify snackbar shows field error
- [ ] 6.3 Manual test: Disconnect network and save → verify network error message in snackbar
- [ ] 6.4 Manual test: Verify form data is preserved after snackbar error message
- [ ] 6.5 Manual test: Verify snackbar auto-dismiss timing is appropriate for different error types
- [ ] 6.6 Smoke test: Verify successful campaign creation still works after changes
- [ ] 6.7 Smoke test: Verify successful campaign edit still works after changes
- [ ] 6.8 Smoke test: Verify multi-field validation errors are formatted correctly in snackbar
