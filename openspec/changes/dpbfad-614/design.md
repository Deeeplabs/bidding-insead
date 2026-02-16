## Context

The INSEAD bidding system currently lacks comprehensive input validation for Credit and Bidding Capital fields across campaign and module configuration forms. The existing validation in `src/src/campaign-management/validation.ts` only provides basic Yup.number() validation with min/max comparison but allows:

1. **Negative values** - Not prevented, which breaks business logic
2. **Multiple decimal places** - Not restricted, causing calculation precision issues  
3. **Unclear error messages** - Non-numeric input protection exists but error messaging is not user-friendly

**Affected Forms:**
- **Campaign Creation/Editing**: `create-campaign.tsx` with fields `min_credits_to_fulfill`, `max_credits_to_fulfill`, `min_capital_granted`, `max_capital_granted`
- **Module/Bidding Round Configuration**: `bidding-configuration.tsx` with fields `min_credits_per_student`, `max_credits_per_student`, `min_capital_per_student`, `max_capital_per_student`

**Current Validation State:**
- Campaign validation: `CampaignCreateValidation` - basic number validation only
- Bidding round validation: `BiddingRoundConfigurationValidation` - basic number validation only
- No custom validation for negative values or decimal precision
- Backend validation may not match frontend restrictions

## Goals / Non-Goals

**Goals:**
- Implement consistent validation rules for all Credit and Bidding Capital inputs
- Provide real-time frontend validation with clear error messages
- Ensure backend validation matches frontend restrictions exactly
- Apply validation to campaign creation/editing and module/bidding round configuration forms
- Maintain backward compatibility for existing valid data
- Support both campaign-level and module-level Credit/Capital fields

**Non-Goals:**
- Changing the underlying data types or database schema
- Modifying business logic for how Credits/Capital are calculated
- Changing the fundamental structure of forms or APIs
- Implementing complex number formatting (currency symbols, thousands separators)

## Decisions

**Frontend Validation Strategy**

Use existing validation patterns in each frontend application:
- **bidding-admin**: Extend Formik + Yup validation schemas with custom validation functions
- **bidding-web**: Enhance custom validation hooks and Ant Design form validation
- Implement real-time validation on input change events
- Use consistent error message components

**Backend Validation Strategy**

Extend existing DTO validation patterns:
- Add custom validation constraints to Request DTOs
- Use Symfony Validator component with custom constraint classes
- Ensure validation error responses follow existing API error format
- Apply validation at the DTO level before business logic execution

**Validation Rules Implementation**

Create a shared validation logic approach:
- Numeric validation: Ensure input can be parsed as a number
- Non-negative validation: Ensure value >= 0
- Decimal precision validation: Ensure at most 1 decimal place
- Edge case handling: Zero values, empty values, malformed input

**Error Messaging Strategy**

Standardize error messages across frontend and backend:
- "Only numeric values are allowed"
- "Negative values are not allowed"  
- "Only one decimal place is allowed"

## Risks / Trade-offs

- **[Risk] Breaking existing workflows** if users were relying on lenient validation → Mitigation: Clear error messages and graceful degradation
- **[Risk] Performance impact** of real-time validation → Mitigation: Efficient validation functions, debounced validation where appropriate
- **[Trade-off] Strict validation vs user flexibility** → Chose strict validation for data integrity over permissive input

## Technical Implementation

### Frontend - bidding-admin (Next.js + Formik + Yup)

**File locations:**
- `src/src/campaign-management/validation.ts` - Campaign and bidding round validation schemas
- `src/components/forms/ValidationMessages.tsx` - Shared error message components

**Implementation approach:**
```typescript
// Custom Yup validation methods
Yup.addMethod(Yup.number, 'positiveDecimal', function() {
  return this.test('positiveDecimal', 'Negative values are not allowed', function(value) {
    return value === undefined || value === null || value >= 0;
  }).test('oneDecimalPlace', 'Only one decimal place is allowed', function(value) {
    if (value === undefined || value === null) return true;
    const decimal = value.toString().split('.')[1];
    return !decimal || decimal.length <= 1;
  });
});

// Usage in validation schemas
const campaignValidationSchema = Yup.object({
  min_credits_to_fulfill: Yup.number().positiveDecimal().required(),
  max_credits_to_fulfill: Yup.number().positiveDecimal().required(),
  min_capital_granted: Yup.number().positiveDecimal().required(),
  max_capital_granted: Yup.number().positiveDecimal().required(),
});

const biddingRoundValidationSchema = Yup.object({
  min_credits_per_student: Yup.number().positiveDecimal().required(),
  max_credits_per_student: Yup.number().positiveDecimal().required(),
  min_capital_per_student: Yup.number().positiveDecimal().required(),
  max_capital_per_student: Yup.number().positiveDecimal().required(),
});
```

### Frontend - bidding-web (Next.js + Ant Design + Custom Hooks)

**No changes required** - Students only view capital/credit values that come from the API. They don't input these values directly. The bidding-web application only displays:
- `capital_granted`, `capital_spent`, `capital_left` (read-only display)
- `credits_taken`, `credits_maximum`, `credits_minimum` (read-only display)
- Students only input bid points for individual courses, not total capital/credit amounts

### Backend - bidding-api (Symfony + Doctrine)

**File locations:**
- `src/Dto/Request/` - Update relevant Request DTOs
- `src/Validator/Constraints/` - Custom validation constraint classes
- `src/Validator/` - Custom validator classes

**Implementation approach:**

1. **Create custom validation constraint:**
```php
// src/Validator/Constraints/PositiveDecimal.php
#[Attribute(Attribute::TARGET_PROPERTY)]
class PositiveDecimal extends Constraint
{
    public string $negativeMessage = 'Negative values are not allowed';
    public string $decimalMessage = 'Only one decimal place is allowed';
    public string $numericMessage = 'Only numeric values are allowed';
    
    public function validatedBy(): string
    {
        return PositiveDecimalValidator::class;
    }
}
```

2. **Create validator class:**
```php
// src/Validator/PositiveDecimalValidator.php
class PositiveDecimalValidator extends ConstraintValidator
{
    public function validate(mixed $value, Constraint $constraint): void
    {
        if (null === $value || '' === $value) {
            return;
        }
        
        // Check if numeric
        if (!is_numeric($value)) {
            $this->context->buildViolation($constraint->numericMessage)
                ->addViolation();
            return;
        }
        
        $numValue = (float) $value;
        
        // Check if negative
        if ($numValue < 0) {
            $this->context->buildViolation($constraint->negativeMessage)
                ->addViolation();
            return;
        }
        
        // Check decimal places
        $decimal = explode('.', (string) $value)[1] ?? null;
        if ($decimal && strlen($decimal) > 1) {
            $this->context->buildViolation($constraint->decimalMessage)
                ->addViolation();
        }
    }
}
```

3. **Apply to DTOs:**
```php
// src/Dto/Request/Campaign/CreateCampaignRequest.php
class CreateCampaignRequest implements DtoInterface
{
    #[PositiveDecimal]
    public ?int $minCreditsToFulfill;
    
    #[PositiveDecimal]
    public ?int $maxCreditsToFulfill;
    
    #[PositiveDecimal]
    public ?int $minCapitalGranted;
    
    #[PositiveDecimal]
    public ?int $maxCapitalGranted;
}
```

### API Error Response Format

Follow existing API error response pattern:
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

## Testing Strategy

**Note: All automated testing has been removed from this project. The validation implementation will rely on manual verification and runtime validation.**

### Frontend Testing
- ~~Unit tests for validation utility functions (bidding-admin only)~~ - **REMOVED**
- ~~Component tests for enhanced input components (bidding-admin only)~~ - **REMOVED**
- ~~Integration tests for form submission with invalid data (bidding-admin only)~~ - **REMOVED**

### Backend Testing  
- ~~Unit tests for custom validation constraints~~ - **REMOVED**
- ~~Integration tests for API endpoints with invalid input~~ - **REMOVED**
- ~~Edge case testing (zero values, empty strings, malformed numbers)~~ - **REMOVED**

### Cross-Application Testing
- ~~Verify consistent validation behavior between admin frontend and backend~~ - **REMOVED**
- ~~Test API error responses match frontend validation expectations~~ - **REMOVED**
- ~~Manual testing of campaign and module configuration forms~~ - **REMOVED**

**Verification Approach:**
- Manual testing during development and deployment
- Runtime validation feedback from users
- Code review of validation logic
- Production monitoring for validation-related issues
