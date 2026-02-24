## Why

When a Programme Manager (PM) duplicates a campaign, three critical issues occur:

1. **Frozen Simulation Component**: Upon campaign duplication where phase timelines fall back to null dates, the Simulation section becomes locked and falsely reports "Simulation Completed", preventing PMs from running the simulator or rebalancing class capacities.

2. **Missing Reference Integrity**: The Detail Bidding display falls into "undefined status" mapping loop because the newly duplicated Campaign Phase does not know what module block it belongs to, resulting in corrupted object graphs and broken UI.

3. **Module Detail 404 Errors**: After campaign duplication, when PM updates module dates, the backend fails to mark the configuration as "active". The module remains flagged as inactive internally, causing 404 errors when loading bidding details.

## What Changes

### Backend Fixes Required

1. **Fix Simulation Date Evaluation** (SimulationDashboardService.php)
   - Update `$isActive` conditional logic to handle null/missing `$startDate` and `$endDate` values
   - Default to `CLOSE` ("Not Started") instead of `COMPLETED` for null dates

2. **Fix Campaign Duplication Reference Integrity** (CampaignDuplicationService.php)
   - Explicitly link new `CampaignPhaseConfig` to cloned `CampaignModule` during duplication
   - Call `$newPhaseConfig->setCampaignModule($newExecution->getCampaignModule())` in the phase config loop

3. **Fix Module Activation on Date Edit** (CampaignPhaseService.php)
   - Call `$phaseConfig->setIsActive(true)` in `saveModuleConfigByCampaignAndModule`
   - Call `$phaseConfig->setIsActive(true)` in `configureAddDropDeadline`
   - This ensures duplicated modules become active when PM edits their configuration

4. **Fix Auto-Activation for Existing Campaigns** (AdminCampaignDetailService.php)
   - Add auto-activation logic that activates phase configs with dates but isActive=false
   - This handles existing duplicated campaigns saved before the fix was deployed

5. **Fix Phase Config Status Mapping** (`CampaignResponse.php` & `CampaignDetailResponse.php`)
   - Change `is_active` in the `phase_configs` array to evaluate to `'OPEN'`, `'COMPLETED'`, or `'CLOSE'` based on the configured dates instead of returning the boolean property directly.

## Capabilities

### New Capabilities
- None - this is a bug fix

### Modified Capabilities
- `campaign-management`: Campaign duplication and status handling

## Impact

### Affected Components

**Backend (bidding-api) only:**
- `src/Domain/Simulation/Service/SimulationDashboardService.php` - getModulePhaseDetails() method
- `src/Domain/Campaign/Campaign/CampaignDuplicationService.php` - duplicateCampaign() method
- `src/Domain/Campaign/Campaign/CampaignPhaseService.php` - saveModuleConfigByCampaignAndModule() and configureAddDropDeadline() methods
- `src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php` - getCampaignDetail() method with auto-activation

**Frontend (bidding-admin)**
- No changes needed - issue is on backend API side

## Root Cause Summary

1. **Date Default Evaluation**: Unguarded logic in `SimulationDashboardService::getModulePhaseDetails()` assumes null endDate = COMPLETED, locking the UI
2. **Orphaned Database Relations**: Campaign duplication omits linking new CampaignPhaseConfig to its CampaignModule
3. **Inactive Duplicated Modules**: Date/config updates don't set isActive=true, causing 404 errors
4. **Existing Duplicated Campaigns**: For campaigns saved before the fix was deployed, isActive remains false

## Backward Compatibility

This fix addresses state interpretation logic only:
- No changes to user access or capabilities
- Preserves existing functionality for non-duplicated campaigns
- No migration required - logic fix only
