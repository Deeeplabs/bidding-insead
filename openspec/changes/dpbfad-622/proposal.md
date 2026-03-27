## Why

Currently, students can access pre-bidding URLs for campaigns belonging to a different Programme/Promotion by copying and pasting the URL directly. This is a security vulnerability - a student assigned to EMBA can view MBA campaign details by navigating directly to the MBA pre-bidding URL. The system only validates programme access during UI navigation but not on direct URL access.

## What Changes

- Add programme and promotion validation on every pre-bidding page access in bidding-web
- Add server-side validation in bidding-api to check user's Programme/Promotion matches the campaign
- Redirect unauthorized users to Access Denied page
- Ensure no campaign data is exposed before authorization check

## Capabilities

### New Capabilities
- `pre-bidding-access-control`: Validates Programme and Promotion authorization before displaying pre-bidding campaign details

### Modified Capabilities
- (none - this is a new security check, not modifying existing behavior)

## Impact

**Affected apps:**
- `bidding-web` - Student portal (frontend validation + redirect)
- `bidding-api` - Backend API (server-side validation)

**Affected code:**
- CampaignController or PreBiddingController in bidding-api (add authorization check)
- Pre-bidding page component in bidding-web (add auth validation on mount)

**Migration requirements:**
- (none - this is a logic change, not database change)

**Backward compatibility:**
- Existing authorized users will not be affected
- Only unauthorized access is being blocked
