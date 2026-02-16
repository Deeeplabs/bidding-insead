# Add comprehensive input validation for Credit and Bidding Capital fields

## Problem
**Jira**: https://insead.atlassian.net/browse/DPBFAD-614

The INSEAD bidding system lacked comprehensive input validation for Credit and Bidding Capital fields on the backend. While frontend validation provided some protection, the API endpoints did not enforce consistent validation rules, allowing:

1. **Negative values** - Not prevented at the API level, which could break business logic
2. **Multiple decimal places** - Not restricted at the API level, causing calculation precision issues  
3. **Non-numeric input** - Inconsistent validation between frontend and backend
4. **Missing validation** - Some Credit/Capital fields lacked proper validation constraints

**Affected API Endpoints:**
- Campaign update and creation endpoints with Credit/Capital fields
- Campaign deployment from presets with Credit/Capital configuration
- Module/bidding round configuration endpoints with Credit/Capital fields

## Solution

### 1. Custom Validation Constraint

**PositiveDecimal Constraint Class**
- Created `src/Validator/Constraints/PositiveDecimal.php` 
- Defines three validation error messages:
  - `numericMessage`: "Only numeric values are allowed"
  - `negativeMessage`: "Negative values are not allowed"
  - `decimalMessage`: "Only one decimal place is allowed"
- Uses PHP 8 attributes for constraint declaration

**PositiveDecimal Validator Class**
- Created `src/Validator/PositiveDecimalValidator.php`
- Implements comprehensive validation logic:
  - Numeric validation using `is_numeric()`
  - Negative value validation with float conversion
  - Decimal place validation using string splitting
  - Proper handling of null/empty values (delegates to required validation)

### 2. DTO Validation Updates

**Campaign Request DTOs**
- `UpdateCampaignRequest.php`: Applied `#[PositiveDecimal]` to 4 fields:
  - `minCreditsToFulfill`, `maxCreditsToFulfill`
  - `minCapitalGranted`, `maxCapitalGranted`
- `DeployPresetRequest.php`: Applied `#[PositiveDecimal]` to same 4 fields

**Module Configuration Request DTOs**
- `BiddingRoundConfigRequest.php`: Applied `#[PositiveDecimal]` to 4 fields:
  - `min_credits_per_student`, `max_credits_per_student`
  - `min_capital_per_student`, `max_capital_per_student`

**Additional DTOs**
- `UpdatePromotionSettingRequest.php`: Applied validation to relevant Credit/Capital fields

### 3. Validation Logic Implementation

**Three-Tier Validation**
1. **Numeric Check**: Ensures input can be parsed as a number using `is_numeric()`
2. **Non-Negative Check**: Validates that the numeric value is >= 0
3. **Decimal Precision Check**: Limits to maximum 1 decimal place using string parsing

**Error Handling**
- Early return on validation failures to prevent multiple error messages
- Proper Symfony Validator integration with violation building
- Consistent error message format across all validation failures

## Files Changed

| File | Change |
|------|--------|
| `src/Validator/Constraints/PositiveDecimal.php` | New custom validation constraint class |
| `src/Validator/PositiveDecimalValidator.php` | New custom validator implementation |
| `src/Controller/Api/Campaign/UpdateCampaignRequest.php` | Added PositiveDecimal to 4 Credit/Capital fields |
| `src/Controller/Api/Campaign/DeployPresetRequest.php` | Added PositiveDecimal to 4 Credit/Capital fields |
| `src/Controller/Api/Campaign/ModuleConfig/BiddingRoundConfigRequest.php` | Added PositiveDecimal to 4 Credit/Capital fields |
| `src/Controller/Api/Promotion/Update/UpdatePromotionSettingRequest.php` | Added PositiveDecimal to relevant fields |

## Validation Rules Implemented

### API-Level Validation Rules
1. **Numeric validation** - Rejects non-numeric input with clear error message
2. **Non-negative validation** - Prevents negative numbers that break business logic
3. **Decimal precision validation** - Limits to one decimal place for consistency
4. **Null/empty handling** - Properly handles optional fields while maintaining required field validation

### Error Response Format
Follows existing API error response pattern:
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": {
      "minCreditsToFulfill": ["Only numeric values are allowed"],
      "maxCapitalGranted": ["Negative values are not allowed"]
    }
  }
}
```

## Impact

- **Enhanced data integrity** - Backend validation prevents invalid data from reaching business logic
- **Consistent validation** - Frontend and backend now enforce identical validation rules
- **Clear error messages** - API responses include specific validation error messages
- **Improved security** - Additional layer of validation prevents malformed data submission
- **Backward compatible** - Existing valid data continues to work; invalid data receives proper error responses

## Testing

Manual verification completed:
- All Credit/Capital fields in campaign endpoints validate correctly
- All Credit/Capital fields in module configuration endpoints validate correctly
- API error responses include appropriate validation error messages
- Validation works consistently across all affected endpoints
- Form submission with invalid data returns proper 400 Bad Request responses
