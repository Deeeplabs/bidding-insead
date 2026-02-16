## Why

Currently, the system allows invalid input for Credit and Bidding Capital fields across campaign creation/editing and module configuration forms. Users can enter:

1. **Invalid characters** - While InputNumberComponent provides some protection, custom error messages are unclear
2. **Negative values** - Currently allowed by validation schemas, which breaks business logic
3. **Multiple decimal places** - Currently allowed, causing calculation precision issues

The affected fields include:
- **Campaign level**: `min_credits_to_fulfill`, `max_credits_to_fulfill`, `min_capital_granted`, `max_capital_granted`
- **Module/Bidding Round level**: `min_credits_per_student`, `max_credits_per_student`, `min_capital_per_student`, `max_capital_per_student`

Business rules require:
1. Only numeric values are allowed for Credits and Bidding Capital
2. No negative values are permitted
3. Only one decimal place is allowed for precision
4. Validation must be consistent between frontend and backend

## What Changes

- Implement frontend input validation for Credit and Bidding Capital fields with real-time feedback
- Add backend validation rules to enforce the same constraints  
- Update campaign creation/editing forms and module/bidding round configuration forms
- Ensure validation error messages are clear and user-friendly
- Apply validation consistently across all Credit/Capital input fields in both campaign and module contexts

## Capabilities

### New Capabilities

- **input-validation**: Real-time validation for Credit and Bidding Capital fields with proper error messaging and constraint enforcement

### Modified Capabilities

- **campaign-configuration**: Campaign creation/editing forms will have enhanced validation for Credit/Capital fields
- **module-configuration**: Module and bidding round configuration forms will have enhanced validation for Credit/Capital fields
- **api-validation**: Backend endpoints will enforce the same validation rules as frontend

## Impact

- **bidding-admin**: Form validation components and schemas need updates for all Credit/Capital input fields
- **bidding-api**: Request DTOs and validation logic need updates to enforce numeric, non-negative, single-decimal constraints
- **Entities affected**: Campaign, CampaignModule, and any configuration entities with Credit/Capital fields
- **No changes required for bidding-web** - Students only view capital/credit values, they don't input them directly
- **No database schema changes required** - this is purely validation logic enhancement
- **Backward compatibility**: Existing valid data will continue to work; invalid data will be rejected with clear error messages
