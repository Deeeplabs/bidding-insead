# Implementation Tasks

## 1. Fix Campaign Duplication Preset Issue (Issue 1)

- [x] 1.1 Modify CampaignDuplicationService.php to conditionally copy preset based on duplicateCampaignFlow parameter
  - File: `bidding-api/src/Domain/Campaign/Campaign/CampaignDuplicationService.php`
  - Move `$newCampaign->setPreset($sourceCampaign->getPreset())` from line 59 inside the `if ($duplicateCampaignFlow)` block (around line 167)

## 2. Add Campaign Phase Date Validation (Issue 2)

- [x] 2.1 Add validatePhaseDateOverlap method to CampaignValidationService
  - File: `bidding-api/src/Domain/Campaign/Campaign/CampaignValidationService.php`
  - Method should check for overlapping dates between bidding_round and simulation phases
  - Return array of error messages

- [x] 2.2 Call validatePhaseDateOverlap in validateCampaignStart method
  - File: `bidding-api/src/Domain/Campaign/Campaign/CampaignValidationService.php`
  - Add validation call in `validateCampaignStart()` method to prevent campaign from opening with invalid dates

## 3. Fix Preset Edit Validation on Open Campaign (Issue 3)

- [x] 3.1 Modify CampaignService.php to compare existing preset_id with new preset_id
   - File: `bidding-api/src/Domain/Campaign/Campaign/CampaignService.php`
   - Add logic: `if ($campaign->getStatus() === 'open' && $existingPresetId != $newPresetId)`
   - Only block when status is "open" AND preset is actually changing

- [x] 3.2 Test campaign edit on open campaign with same preset_id
   - Endpoint: PUT /v2/api/campaigns/{id}
   - Verify 200 OK when preset_id hasn't changed
   - Verify 400 error when preset_id has changed

## 4. Frontend Integration (Optional - Based on UI Requirements)

- [x] 3.1 Add date validation error display in campaign editing form (bidding-admin)
  - Files: Check `bidding-admin/src/app/(authenticated)/campaign/` for relevant components
  - Display validation errors from API response when saving campaign

- [x] 3.2 Verify preset dropdown behavior in campaign duplication UI
  - Test: When duplicating without campaign flow, preset dropdown should be empty

## 4. Integration Testing

- [x] 4.1 Test campaign duplication API with duplicate_campaign_flow=false
  - Endpoint: POST /v2/api/campaigns/{id}/duplicate
  - Verify response has preset: null

- [x] 4.2 Test campaign open API with overlapping phase dates
  - Endpoint: POST /v2/api/campaigns/{id}/open
  - Verify 400 response with validation error message

## 5. Verification

- [x] 5.1 Final verification - no regressions in existing functionality
- [x] 5.2 Verify all three issues are fixed as per requirements
