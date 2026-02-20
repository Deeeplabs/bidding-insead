## Context

There are **three** bugs in the campaign management system that need to be fixed:

1. **Preset Duplication Issue**: When duplicating a campaign without selecting "Duplicate Campaign Flow", the preset is still being copied from the source campaign. This creates user confusion as they expect an empty preset dropdown.

2. **Missing Date/Time Clash Validation**: When configuring campaign phases (bidding rounds and simulation), there's no validation to prevent overlapping dates. This causes runtime errors when running the campaign.

3. **Preset Edit Validation Issue**: When editing a campaign that is already "open", the system incorrectly blocks ALL preset changes, even when the preset_id remains the same. This prevents users from saving the campaign even when no actual preset change was made.

## Goals / Non-Goals

**Goals:**
- Fix preset not being cleared during campaign duplication when `duplicateCampaignFlow=false`
- Add date/time overlap validation between bidding rounds and simulation phases
- Fix preset edit validation to allow saving when preset_id hasn't actually changed on open campaigns
- Show clear error messages to users before they can save/proceed

**Non-Goals:**
- No changes to preset management UI or functionality
- No changes to campaign phase sequencing logic
- No database schema changes required

## Decisions

### Issue 1: Preset Duplication Fix

**Decision**: Modify `CampaignDuplicationService.php` to conditionally copy preset based on `duplicateCampaignFlow` parameter.

**Rationale**: The preset is only meaningful when the campaign flow (workflow modules) is being duplicated. The current code always copies the preset on line 59, regardless of the `duplicateCampaignFlow` parameter.

**Implementation**:
- In `CampaignDuplicationService::duplicateCampaign()`, move the `setPreset()` call inside the `if ($duplicateCampaignFlow)` block (around line 167 where campaign flow duplication happens)

### Issue 2: Date/Time Clash Validation

**Decision**: Add validation in `CampaignValidationService` to check for overlapping dates between bidding round and simulation phases.

**Rationale**: 
- `CampaignValidationService` already handles campaign-level validation
- Adding validation here ensures it's checked before campaign can be opened/run
- Validation should check that bidding_round phase end date is before simulation phase start date

**Implementation**:
- Add new method `validatePhaseDateOverlap()` in `CampaignValidationService`
- This method should iterate through campaign modules and check that:
  - Bidding round phases end before simulation phases start
  - No two phases have overlapping date ranges
- Call this validation in `validateCampaignStart()` to prevent campaign from starting with invalid phase dates

### Frontend Changes

**Decision**: Display validation errors in the campaign editing UI when date overlaps are detected.

**Rationale**:
- Users need immediate feedback when configuring phase dates
- Frontend should show the same validation as backend for better UX

**Implementation**:
- Add validation check in the campaign form (likely in `bidding-admin`)
- Display error messages from the API response

## Risks / Trade-offs

1. **Risk**: Changing preset behavior might affect existing workflows
   - **Mitigation**: Only change behavior when `duplicateCampaignFlow=false`. When true, behavior remains the same.

2. **Risk**: Date validation might block campaigns with legitimate configurations
   - **Mitigation**: Only block if there's an actual overlap. Allow gaps between phases.

3. **Risk**: Backend validation might not be called in all code paths
   - **Mitigation**: Ensure validation is called in `validateCampaignStart()` which is the gate for opening campaigns

### Issue 3: Preset Edit Validation Fix

**Decision**: Modify `CampaignService.php` to compare existing preset_id with new preset_id before blocking changes.

**Rationale**: The current code blocks ALL preset changes on open campaigns, but this is too restrictive. Users should be able to save/edit other campaign settings without changing the preset.

**Implementation**:
- In the campaign update/edit method, add logic to compare `$existingPresetId` with `$newPresetId`
- Only block the edit if: `campaign status is "open"` AND `existing preset_id != new preset_id`
- If preset_id remains the same, allow the edit to proceed

## Migration Plan

1. Deploy API changes first (validation + duplication fix)
2. Deploy Admin UI changes for error display
3. No database migrations needed (validation logic only)

## Open Questions

- Should the validation also check for gaps? (e.g., simulation should start after bidding ends)
- Should there be a minimum gap between phases?
