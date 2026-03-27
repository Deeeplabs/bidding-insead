# Pre-Bidding Programme-Based Access Control

## Problem
Students can access pre-bidding URLs for campaigns belonging to a different Programme/Promotion by copying and pasting the URL directly. The system only validates programme access during UI navigation but not on direct URL access, allowing unauthorized users to view campaign details.

## Goal
Implement programme and promotion validation on every pre-bidding page access:
- Validate that the student's Programme and Promotion match the campaign's Programme and Promotion
- Redirect unauthorized users to Access Denied page
- Ensure no campaign data is exposed before authorization check

## Changes Made (bidding-api)

### 1. `src/Controller/Api/Student/Campaign/StudentActiveCampaignController.php`
- Added Programme/Promotion authorization check in the `getBiddingRoundDetail` method (line ~185)
- Fetches the campaign and compares student's Promotion and Program against campaign's Promotion and Program
- Returns 403 Forbidden if user's Promotion doesn't match Campaign's Promotion
- Returns 403 Forbidden if user's Program doesn't match Campaign's Program
- Handles edge case where Campaign has no Programme assigned (allow access for backward compatibility)

### Authorization Logic
```php
// Check if student's promotion matches campaign's promotion
if ($campaignPromotion !== null && $studentPromotion !== null) {
    if ($campaignPromotion->getId() !== $studentPromotion->getId()) {
        return new JsonResponse([
            'error' => 'Access denied: You do not have permission to view this campaign'
        ], Response::HTTP_FORBIDDEN);
    }
}

// Check if student's program matches campaign's program
if ($campaignProgram !== null && $studentProgram !== null) {
    if ($campaignProgram->getId() !== $studentProgram->getId()) {
        return new JsonResponse([
            'error' => 'Access denied: You do not have permission to view this campaign'
        ], Response::HTTP_FORBIDDEN);
    }
}
```

## API Response Change

For unauthorized access:
```json
HTTP 403 Forbidden
{
  "error": "Access denied: You do not have permission to view this campaign"
}
```

This is an **additive, non-breaking change** - only adds authorization checks to existing endpoints.

## Impact
- No migrations required — authorization logic only
- No breaking changes — authorized users unaffected
- Unauthorized users will be redirected to Access Denied page when accessing campaigns outside their Programme/Promotion
- No campaign data exposed in 403 response

## Testing / Verification Steps
1. As a student with Programme "EMBA", navigate to an MBA campaign pre-bidding URL - should redirect to Access Denied
2. As a student with matching Programme/Promotion, access pre-bidding URL - should succeed
3. API request to `/api/student/active-campaigns/{id}` for unauthorized campaign - should return 403
4. Verify no campaign data in unauthorized response
