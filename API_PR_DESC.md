# Fix Campaign Status and Simulation Duplication Bug

## Problem

When a Programme Manager (PM) duplicates a campaign, they encounter issues when trying to proceed with configuration or operations within the new campaign.

1. **Frozen Simulation Component**: Upon campaign duplication, where phase timelines naturally fall back to empty inputs (`null`), the application's Simulation section becomes functionally locked and falsely reports "Simulation Completed". This prevents PMs from running the simulator or re-adjusting class capacities—a critical component of the platform's rebalancing feature.
2. **Missing Reference Integrity**: The Detail Bidding display falls into an "undefined status" mapping loop because the newly duplicated Campaign Phase does not know what module block it belongs to, resulting in corrupted object graphs and broken UI operations.
3. **Module Detail 404 Errors**: After a campaign is duplicated, if a Programme Manager updates the module dates to configure it, the backend fails to mark the underlying configuration as "active". Because the module remains flagged as inactive internally, subsequent attempts to load its bidding details result in a 404 "Unable to load bidding details" error.

### Root Cause Analysis

1. **Date Default Evaluation**: In `SimulationDashboardService::getModulePhaseDetails()`, an unguarded logic block assumed that any phase lacking an `endDate` timestamp has inherently elapsed, and thus assigned it the status of `COMPLETED`. Since the Simulation frontend reacts to `COMPLETED` phases by locking all buttons (runs, commits, rollbacks, and capacity adjusting), the new, unstarted phase became permanently inoperable.
2. **Orphaned Database Relations**: During campaign duplication inside `CampaignDuplicationService::duplicateCampaign()`, the loop properly initialized new `CampaignPhaseConfig` objects. However, it omitted explicitly linking the new configuration back to its cloned `CampaignModule`. Because the foreign key remained null, backend queries failed to retrieve the necessary parent contexts, causing the whole entity block to appear "undefined" on the client interface.
3. **Inactive Duplicated Modules**: Duplicated `CampaignPhaseConfig` records are cloned with `isActive = false` by default. When these modules' dates and configurations are updated by the user via the frontend, the endpoints (specifically `saveModuleConfigByCampaignAndModule` and `configureAddDropDeadline` in `CampaignPhaseService.php`) updated the database fields but missed calling `setIsActive(true)`. Consequently, the module remained inactive and inaccessible.
4. **Existing Duplicated Campaigns**: For campaigns that were duplicated and saved before the fix was deployed, the `isActive` flag remained `false` in the database, causing 404 errors when accessing the detail endpoint.
5. **Phase Configs Status Null String Matching**: `is_active` parameter within the `phase_configs` list payload of list/detail campaign endpoints formerly output raw DB boolean states (or nulls). The UI maps active statuses using explicit `OPEN`, `COMPLETED`, `CLOSE` identifiers for completed/unstarted phases, creating a mismatch that disrupted module displays in the frontend dashboard.
6. **Mismatched Campaign List Status**: On the campaign list screen, a module might incorrectly appear as grey/`CLOSE`, but when clicking into the detail view, it successfully displays `COMPLETED`. The campaign list module status was being overwritten by its last chronological phase config (e.g. `results_publication` with no dates), causing the completed `bidding_round` phases to be ignored in the returned DTO.

## Solution

The solution stabilizes the Simulation module evaluation logic strictly for null/missing dates, restores complete referential integrity during duplication, auto-activates phase configs for existing duplicated campaigns, and safely evaluates status strings based on chronological validity.

### Changes Made

**Modified File 1:** `src/Domain/Simulation/Service/SimulationDashboardService.php`
- Updated the `$isActive` conditional logic to responsibly evaluate missing/empty `$startDate` and `$endDate` values. Now, any phase missing dates correctly defaults back to `CLOSE` (which safely translates to "Not Started" for the UI framework), bringing the frozen simulation screen back to a normal, unstarted configuration loop.

**Modified File 2:** `src/Domain/Campaign/Campaign/CampaignDuplicationService.php`
- Explicitly injected `$newPhaseConfig->setCampaignModule($newExecution->getCampaignModule())` deep within the nested phase config loop. This guarantees the freshly cloned phase accurately records its module-level foreign key to the database, so dashboard mappings successfully resolve.

**Modified File 3:** `src/Domain/Campaign/Campaign/CampaignPhaseService.php`
- Added `$phaseConfig->setIsActive(true);` inside `saveModuleConfigByCampaignAndModule` and `configureAddDropDeadline` to ensure that previously inactive (duplicated) modules are properly activated when their configuration or dates are saved by the PM.

**Modified File 4:** `src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php`
- Added `autoActivatePhaseConfig()` method that auto-activates phase configs when accessing the detail endpoint. This fix handles existing duplicated campaigns that were saved before the code fix was deployed - if a phase config has dates set but `isActive` is false, it automatically activates and persists the change.

**Modified File 5:** `src/Controller/Api/Campaign/CampaignResponse.php` & `src/Controller/Api/Campaign/CampaignDetailResponse.php`
- Adjusted `phase_configs` iterator to interpret the phase config's `startDate` and `endDate` boundaries relative to the present time, emitting an unambiguous `'OPEN'`, `'COMPLETED'`, or `'CLOSE'` string under the `is_active` key instead of relying on its database boolean properties.

**Modified File 6:** `src/Domain/Campaign/Campaign/CampaignService.php`
- Fixed the module-level `$is_active` aggregation logic inside `listCampaignsDto()`. Instead of irresponsibly overwriting the module's generic status pointer inside the phase config loop, the builder now collects all calculated config status strings and accurately derives the active priority constraint (`OPEN` takes priority, then `COMPLETED`, lastly `CLOSE`). This ensures that list displays faithfully reflect the same completion data computed by the detailed admin access routes.

## Files Changed

| File | Change |
|------|--------|
| `src/Domain/Simulation/Service/SimulationDashboardService.php` | Change `$isActive` evaluations to safely fallback to `CLOSE` upon encountering `null` dates. |
| `src/Domain/Campaign/Campaign/CampaignDuplicationService.php` | Invoke `setCampaignModule` manually during `CampaignPhaseConfig` duplication cloning. |
| `src/Domain/Campaign/Campaign/CampaignPhaseService.php` | Enforce `$phaseConfig->setIsActive(true)` when editing existing modules/dates during `saveModuleConfigByCampaignAndModule` and `configureAddDropDeadline`. |
| `src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php` | Auto-activate phase configs that have dates but `isActive=false` when accessing the detail endpoint. |
| `src/Controller/Api/Campaign/CampaignResponse.php` | Check phase config `startDate` and `endDate` against the current time and return string `'OPEN'`, `'COMPLETED'`, or `'CLOSE'`. |
| `src/Controller/Api/Campaign/CampaignDetailResponse.php` | Check phase config `startDate` and `endDate` against the current time and return string `'OPEN'`, `'COMPLETED'`, or `'CLOSE'`. |
| `src/Domain/Campaign/Campaign/CampaignService.php` | Add nested aggregation constraints to prevent list campaigns from hiding accurately `COMPLETED` phase modules. |

## Impact

*   **UI Reactivation:** Simulations perfectly identify as "Not Started", keeping the Rebalancing pie chart UI responsive and all operation buttons operable.
*   **Data Integrity:** Duplicated campaigns construct structurally perfect internal maps, fully solving any instances of "undefined statuses".
*   **Backward Compatibility:** Existing duplicated campaigns (saved before this fix) work correctly without requiring re-saving.
*   **Security:** This patch only addresses state interpretation logic without escalating user access or capabilities outside existing constraints.

## Testing

Verified through functional scenarios:
- [x] Tested initiating a "Duplicate Campaign" flow and ensured the new campaign modules are constructed cleanly in the DB with no null `campaign_module_id` relations on their phase configs.
- [x] Verified by inspecting the Simulation step and confirming it displays "Not Started" and the manual capacity Rebalancing interface components click open normally.
- [x] Verified regressions on `activatePhase` capabilities natively remain undisturbed.
- [x] Verified that setting dates for a duplicated module properly marks it as active and allows the module detail endpoint to load without a 404 error.
- [x] Verified that existing duplicated campaigns (saved before fix) work correctly via auto-activation.
