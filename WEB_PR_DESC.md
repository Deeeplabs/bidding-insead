# Pre-Bidding Programme-Based Access Control

## Problem
Students can access pre-bidding URLs for campaigns belonging to a different Programme/Promotion by copying and pasting the URL directly. The system only validates programme access during UI navigation but not on direct URL access, allowing unauthorized users to view campaign details.

## Goal
Implement programme and promotion validation on every pre-bidding page access:
- Validate that the student's Programme and Promotion match the campaign's Programme and Promotion
- Redirect unauthorized users to Access Denied page
- Ensure no campaign data is rendered before authorization check

## Changes Made (bidding-web)

### 1. New Reusable Hook: `src/features/bidding/hooks/use-authorization-check.ts`
- Created a reusable hook for handling 403 Forbidden authorization checks
- Automatically redirects to unauthorized page when API returns "Access denied" error
- Can be reused across all campaign-related pages

### 2. Updated Page Components
Added authorization validation on page mount to the following pages:
- `src/features/bidding/pages/PreBiddingPage.tsx`
- `src/features/bidding/pages/EnrolledPage.tsx`
- `src/features/bidding/pages/BidSubmissionPage.tsx`
- `src/features/bidding/pages/AddDropPage.tsx`

### Usage Example
```tsx
import { useAuthorizationCheck } from '../hooks/use-authorization-check';

const { data, isPending, error } = useQuery(...);

// Simply call the hook - it handles the redirect automatically
useAuthorizationCheck({ isPending, error });
```

### Implementation Details
- Uses existing API error handling - checks if error message contains "Access denied"
- Redirects to `/unauthorized?error_code=UNAUTHORIZED_PROGRAMME_MISMATCH`
- Ensures no campaign data is rendered before authorization check

## UI Flow

### Authorized User
1. Navigate to pre-bidding URL
2. System validates Programme/Promotion match
3. Campaign details displayed

### Unauthorized User
1. Navigate to pre-bidding URL (via direct URL paste)
2. API returns 403 Forbidden
3. Frontend detects "Access denied" error
4. Redirect to Access Denied page with message

## Impact
- No breaking changes - authorized users unaffected
- Unauthorized users will see Access Denied page
- Improved security - prevents unauthorized access to campaign data

## Testing / Verification Steps
1. As a student with Programme "EMBA", paste MBA campaign pre-bidding URL - should redirect to Access Denied page
2. As a student with matching Programme/Promotion, access pre-bidding URL - should display campaign normally
3. Verify Access Denied page shows appropriate message
4. Verify no campaign data visible on Access Denied page
