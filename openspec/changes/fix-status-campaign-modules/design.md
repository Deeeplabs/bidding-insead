## Context

The INSEAD bidding system has bugs in campaign duplication that cause four main issues:
1. Frozen Simulation UI showing "Simulation Completed" when dates are null
2. Missing reference integrity between duplicated CampaignPhaseConfig and CampaignModule
3. Module detail 404 errors due to isActive not being set after date edits
4. Existing duplicated campaigns (saved before the fix) still have isActive=false causing 404 errors

### Current Architecture

**Key Entities:**
- `CampaignModule` - represents a module (pre-bidding, bidding, simulation, etc.) in a campaign
- `CampaignPhaseConfig` - stores phase timing, dates, and configuration for each module
- `Campaign` - the parent campaign entity

**Key Services:**
- `SimulationDashboardService` - provides simulation dashboard data including phase details
- `CampaignDuplicationService` - handles campaign duplication logic
- `CampaignPhaseService` - handles campaign phase configuration

**Frontend:**
- `bidding-admin/src/components/campaign/` - campaign components
- No frontend changes needed - issue is backend API side

## Goals / Non-Goals

**Goals:**
1. Fix SimulationDashboardService to handle null dates properly
2. Fix CampaignDuplicationService to establish proper FK relationships
3. Fix CampaignPhaseService to set isActive=true when saving config
4. Fix AdminCampaignDetailService to auto-activate phase configs for existing campaigns

**Non-Goals:**
- Changing the overall bidding workflow or phases
- Frontend changes (issue is on backend)
- Adding new features

## Decisions

### Decision 1: Null Date Handling in Simulation

**Current Behavior:**
- In `SimulationDashboardService::getModulePhaseDetails()`, missing endDate causes status to be set to COMPLETED
- This locks all simulation buttons (run, commit, rollback, rebalancing)

**Fix:**
- Update logic to default to CLOSE (Not Started) when startDate or endDate is null
- This allows the simulation UI to function normally

**File:** `src/Domain/Simulation/Service/SimulationDashboardService.php`

### Decision 2: Foreign Key Relationship During Duplication

**Current Behavior:**
- CampaignDuplicationService creates new CampaignPhaseConfig objects
- But it doesn't call setCampaignModule() to link them to the new CampaignModule
- This causes "undefined status" in the UI

**Fix:**
- Add explicit call to `$newPhaseConfig->setCampaignModule($newExecution->getCampaignModule())` in the phase config duplication loop

**File:** `src/Domain/Campaign/Campaign/CampaignDuplicationService.php`

### Decision 3: Module Activation on Save

**Current Behavior:**
- When PM edits dates on a duplicated module, the endpoints update the database
- But they don't set isActive = true
- This causes 404 errors when loading the module detail

**Fix:**
- Add `$phaseConfig->setIsActive(true)` to both:
  - `saveModuleConfigByCampaignAndModule()`
  - `configureAddDropDeadline()`

**File:** `src/Domain/Campaign/Campaign/CampaignPhaseService.php`

### Decision 4: Auto-Activate Phase Config for Existing Campaigns

**Current Behavior:**
- Even with the fixes above, existing campaigns that were duplicated BEFORE the fix was deployed still have isActive=false
- This causes 404 errors when accessing the module detail endpoint

**Fix:**
- Add `autoActivatePhaseConfig()` method in AdminCampaignDetailService
- Before returning phase details, check if phase configs have dates set but isActive=false
- Auto-activate them on-the-fly when the detail endpoint is accessed

**File:** `src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php`

### Decision 5: Fix Campaign API Phase Config Status Mapping

**Current Behavior:**
- In the `phase_configs` list within the Campaign list API and detail API responses, `is_active` returns the raw database boolean value (or null).
- Frontend expects this to be evaluated dynamically as `'OPEN'`, `'COMPLETED'`, or `'CLOSE'` based on whether the dates have passed.

**Fix:**
- Update `CampaignResponse.php` and `CampaignDetailResponse.php`.
- Check the phase config `startDate` and `endDate` against the current time and return the string `'OPEN'`, `'COMPLETED'`, or `'CLOSE'` correspondingly.

**File:** `src/Controller/Api/Campaign/CampaignResponse.php` and `src/Controller/Api/Campaign/CampaignDetailResponse.php`

## Risks / Trade-offs

### Risk 1: Regression on Active Campaigns
- **Risk**: Changes might affect existing campaigns that use these services
- **Mitigation**: The logic only affects null date handling and explicit save operations
- **Testing**: Verify existing campaigns still work correctly

### Risk 2: Data Integrity
- **Risk**: Changing isActive might affect campaigns that should remain inactive
- **Mitigation**: Only setting to true when PM explicitly saves configuration

### Risk 3: No Migration Required
- These are logic-only changes, no database schema changes needed

## Migration Plan

1. **Code Changes** (3 files):
   - SimulationDashboardService.php - null date handling
   - CampaignDuplicationService.php - FK relationship
   - CampaignPhaseService.php - isActive flag

2. **Testing**:
   - Test campaign duplication flow
   - Verify simulation shows "Not Started" with null dates
   - Verify module detail loads without 404
   - Verify existing functionality is not affected

3. **Rollback**:
   - Simple code revert if issues arise
   - No data migration to roll back

## Open Questions

None - root cause and solution are now clear from the provided analysis.
