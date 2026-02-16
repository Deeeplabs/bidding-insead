# Add comprehensive input validation for Credit and Bidding Capital fields

## Problem
**Jira**: https://insead.atlassian.net/browse/DPBFAD-614

The INSEAD bidding system lacked comprehensive input validation for Credit and Bidding Capital fields across campaign and module configuration forms. The existing validation only provided basic Yup.number() validation with min/max comparison but allowed:

1. **Negative values** - Not prevented, which breaks business logic
2. **Multiple decimal places** - Not restricted, causing calculation precision issues  
3. **Unclear error messages** - Non-numeric input protection existed but error messaging was not user-friendly

**Affected Forms:**
- **Campaign Creation/Editing**: `create-campaign.tsx` with fields `min_credits_to_fulfill`, `max_credits_to_fulfill`, `min_capital_granted`, `max_capital_granted`
- **Module/Bidding Round Configuration**: `bidding-configuration.tsx` with fields `min_credits_per_student`, `max_credits_per_student`, `min_capital_per_student`, `max_capital_per_student`

## Solution

### 1. Enhanced Frontend Validation

**Custom Yup Validation Method**
- Added `positiveDecimal()` custom method to Yup in `validation.ts`
- Implements three validation rules:
  - Numeric validation: "Only numeric values are allowed"
  - Non-negative validation: "Negative values are not allowed"  
  - Decimal precision: "Only one decimal place is allowed"

**Updated Validation Schemas**
- `CampaignCreateValidation`: Applied `positiveDecimal()` to all 4 Credit/Capital fields
- `BiddingRoundConfigurationValidation`: Applied `positiveDecimal()` to all 4 Credit/Capital fields

**Enhanced Input Component**
- Created `NumericInput.tsx` component with real-time validation
- Provides immediate feedback as users type
- Integrated with existing Formik forms
- Uses Ant Design InputNumber with precision=1 and step=0.1

### 2. Form Integration

**Campaign Creation/Editing Forms**
- Updated `create-campaign.tsx` to use `NumericInput` for all Credit/Capital fields
- Real-time validation feedback with clear error messages
- Maintains existing form structure and behavior

**Module/Bidding Round Configuration Forms**
- Updated `bidding-configuration.tsx` to use `NumericInput` for all Credit/Capital fields
- Consistent validation behavior across all forms
- Proper error display and form submission blocking

## Files Changed

| File | Change |
|------|--------|
| `src/src/campaign-management/validation.ts` | Added custom `positiveDecimal()` Yup method; updated validation schemas |
| `src/components/forms/NumericInput.tsx` | New enhanced input component with real-time validation |
| `src/components/campaign/create-campaign.tsx` | Updated to use NumericInput for Credit/Capital fields |
| `src/components/preset/configuration/bidding-configuration.tsx` | Updated to use NumericInput for Credit/Capital fields |

## Validation Rules Implemented

### Input Validation Rules
1. **Numeric-only input** - Rejects letters, symbols, and other non-numeric characters
2. **Non-negative values** - Prevents negative numbers that break business logic
3. **Single decimal place** - Limits precision to one decimal place for calculation consistency
4. **Real-time feedback** - Immediate error messages as users type

### Error Messages
- "Only numeric values are allowed"
- "Negative values are not allowed"
- "Only one decimal place is allowed"

## Impact

- **Improved data integrity** - Prevents invalid Credit/Capital values from being submitted
- **Better user experience** - Clear, immediate error messages guide users to correct input
- **Consistent validation** - Same rules applied across all Credit/Capital fields
- **Real-time feedback** - Users see errors immediately instead of after form submission
- **Backward compatible** - Existing valid data continues to work unchanged

## Testing

Manual verification completed:
- Campaign creation/editing forms validate all Credit/Capital fields correctly
- Module/bidding round configuration forms validate all Credit/Capital fields correctly
- Real-time validation provides immediate feedback
- Error messages are clear and helpful
- Form submission is blocked until all values are valid
