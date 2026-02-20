# Fix Duplicate Preset and Edit Campaign Issues

## Problem

Three bugs were identified in the campaign management system:

### Issue 1: Preset Duplication Issue
**Jira**: (to be added)

When duplicating a campaign without selecting "Duplicate Campaign Flow", the preset was still being copied from the source campaign. This created user confusion as they expected an empty preset dropdown.

### Issue 2: Missing Date/Time Clash Validation
**Jira**: (to be added)

When configuring campaign phases (bidding rounds and simulation), there was no validation to prevent overlapping dates. This caused runtime errors when running the campaign.

### Issue 3: Preset Change Validation on Open Campaign
**Jira**: (to be added)

When editing a campaign that is already "open", the system was incorrectly blocking ALL preset changes, even when the preset_id remained the same. This prevented users from saving the campaign even when no actual preset change was made.

1. **`campaign_student_filters` DB table** — for campaign-level filters
2. **`campaign_phase_configs.module_config` JSON blob** — for per-module filters (e.g., "Pre-bidding ONLY")

### 1. Preset Duplication Fix

**Modified File**: `src/Domain/Campaign/Campaign/CampaignDuplicationService.php`

- Moved the `setPreset()` call inside the `if ($duplicateCampaignFlow)` block
- Previously, preset was always copied regardless of the `duplicateCampaignFlow` parameter
- Now, when `duplicateCampaignFlow=false`, the new campaign will have `preset: null`
- When `duplicateCampaignFlow=true`, behavior remains unchanged (preset is copied)

**Code Change**:
```php
// Before: Always copied preset (line ~59)
$newCampaign->setPreset($sourceCampaign->getPreset());

// After: Conditionally copy preset based on duplicateCampaignFlow parameter
// (moved inside the if ($duplicateCampaignFlow) block around line ~167)
if ($duplicateCampaignFlow) {
    // ... copy campaign flow ...
    $newCampaign->setPreset($sourceCampaign->getPreset());
}
```

### 2. Date/Time Clash Validation

**Modified File**: `src/Domain/Campaign/Campaign/CampaignValidationService.php`

Added new validation method `validatePhaseDateOverlap()`:
- Iterates through campaign modules to check for overlapping dates
- Validates that bidding round phases end before simulation phases start
- Returns array of error messages when overlaps are detected

**Modified**: `validateCampaignStart()` method now calls `validatePhaseDateOverlap()`
- Prevents campaign from opening/starting with invalid phase dates
- Returns 400 Bad Request with validation error when dates overlap

**Validation Logic**:
- Checks that no two phases have overlapping date ranges
- Specifically validates bidding_round end date is before simulation start date
- Allows gaps between phases (no minimum gap required)

### 3. Preset Change Validation Fix

**Modified File**: `src/Domain/Campaign/Campaign/CampaignService.php`

- Added logic to compare existing preset_id with new preset_id before blocking changes
- Now only blocks preset changes when: campaign status is "open" AND preset_id is actually different
- If preset_id remains the same (no actual change), the edit is allowed

**Code Change**:
```php
// Before: Blocked all preset changes when status is 'open'
if ($campaign->getStatus() === 'open') {
    throw new \InvalidArgumentException('Cannot change preset while campaign is open');
}

// After: Only blocks if status is 'open' AND preset is actually changing
$existingPresetId = $campaign->getPreset()?->getId();
$newPresetId = $data['preset_id'];

if ($campaign->getStatus() === 'open' && $existingPresetId != $newPresetId) {
    throw new \InvalidArgumentException('Cannot change preset while campaign is open');
}
```

## Files Changed

| File | Change |
|------|--------|
| `src/Domain/Campaign/Campaign/CampaignDuplicationService.php` | Conditionally copy preset based on duplicateCampaignFlow parameter |
| `src/Domain/Campaign/Campaign/CampaignValidationService.php` | Added validatePhaseDateOverlap method and integration |
| `src/Domain/Campaign/Campaign/CampaignService.php` | Fixed preset change validation to allow same preset on open campaigns |

## API Changes

### Campaign Duplication Endpoint
- **Endpoint**: `POST /v2/api/campaigns/{id}/duplicate`
- **Parameter**: `duplicate_campaign_flow` (boolean)
- **Behavior Change**: When `duplicate_campaign_flow=false`, response now returns `preset: null`

### Campaign Open Endpoint
- **Endpoint**: `POST /v2/api/campaigns/{id}/open`
- **New Validation**: Returns 400 Bad Request if phase dates overlap
- **Error Response**:
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": {
      "phase_dates": ["Phase dates overlap detected"]
    }
  }
}
```

### Campaign Edit Endpoint
- **Endpoint**: `PUT /v2/api/campaigns/{id}`
- **Behavior Change**: When campaign status is "open", preset changes are now allowed if the preset_id remains the same
- **Error Response** (when changing preset on open campaign):
```json
{
  "error": {
    "code": "INVALID_ARGUMENT",
    "message": "Cannot change preset while campaign is open"
  }
}
```

## Impact

- **Fixed preset confusion**: Users no longer see a preset they didn't expect when duplicating without campaign flow
- **Prevented runtime errors**: Date overlap validation prevents campaigns from failing during execution
- **Fixed preset edit blocking**: Users can now save open campaigns without receiving false preset change errors
- **Clear error messages**: Users receive actionable feedback when configuration is invalid
- **Backward compatible**: Existing functionality preserved; only fixes behavior for edge case

## Testing

Verified through integration tests:
- [x] Campaign duplication with `duplicate_campaign_flow=false` returns `preset: null`
- [x] Campaign duplication with `duplicate_campaign_flow=true` copies preset (unchanged behavior)
- [x] Campaign open with overlapping phase dates returns 400 validation error
- [x] Campaign open with valid phase dates works correctly
- [x] Campaign edit on open campaign with same preset_id succeeds (no error)
- [x] Campaign edit on open campaign with different preset_id returns error
- [x] No regressions in existing campaign functionality
