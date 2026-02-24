## 1. Implementation Tasks

### 1.1 Fix Simulation Null Date Handling

- [x] 1.1.1 Modify SimulationDashboardService.php
  - Location: `src/Domain/Simulation/Service/SimulationDashboardService.php`
  - Method: getModulePhaseDetails()
  - Change: Update $isActive conditional to handle null startDate/endDate
  - Default to CLOSE (Not Started) when dates are null instead of COMPLETED

### 1.2 Fix Campaign Duplication Reference Integrity

- [x] 1.2.1 Modify CampaignDuplicationService.php
  - Location: `src/Domain/Campaign/Campaign/CampaignDuplicationService.php`
  - Method: duplicateCampaign()
  - Change: Add `$newPhaseConfig->setCampaignModule($newExecution->getCampaignModule())` in the phase config duplication loop
  - Ensure FK relationship is established for each cloned CampaignPhaseConfig

### 1.3 Fix Module Activation on Save

- [x] 1.3.1 Modify CampaignPhaseService.php - saveModuleConfigByCampaignAndModule
  - Location: `src/Domain/Campaign/Campaign/CampaignPhaseService.php`
  - Method: saveModuleConfigByCampaignAndModule()
  - Change: Add `$phaseConfig->setIsActive(true);` to activate module when saved

- [x] 1.3.2 Modify CampaignPhaseService.php - configureAddDropDeadline
  - Location: `src/Domain/Campaign/Campaign/CampaignPhaseService.php`
  - Method: configureAddDropDeadline()
  - Change: Add `$phaseConfig->setIsActive(true);` to activate module when dates are configured

### 1.4 Fix Auto-Activation for Existing Campaigns

- [x] 1.4.1 Modify AdminCampaignDetailService.php
  - Location: `src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php`
  - Method: getCampaignDetail()
  - Change: Add autoActivatePhaseConfig() method that auto-activates phase configs with dates but isActive=false
  - Handles existing duplicated campaigns that were saved before the fix was deployed

### 1.5 Fix Campaign API Phase Config Status Mapping

- [x] 1.5.1 Modify CampaignResponse.php and CampaignDetailResponse.php
  - Location: `src/Controller/Api/Campaign/CampaignResponse.php` and `src/Controller/Api/Campaign/CampaignDetailResponse.php`
  - Change: Map `is_active` inside the `phase_configs` payload to represent string status ('OPEN', 'COMPLETED', 'CLOSE') based on start/end dates rather than returning the raw database boolean value.

### 1.6 Fix Campaign Module Status Aggregation

- [x] 1.6.1 Modify CampaignService.php - listCampaignsDto
  - Location: `src/Domain/Campaign/Campaign/CampaignService.php`
  - Change: Fix the calculation of the module's `is_active` parameter by correctly aggregating the statuses of all its Phase Configs (OPEN, COMPLETED, CLOSE) rather than blindly overwriting the value with the last phase config's status.

## 2. Testing Tasks

- [x] 2.1 Test campaign duplication flow
  - Create a new campaign by duplicating an existing one
  - Verify new campaign modules have proper FK relationships
  - Verify no null campaign_module_id in database

- [x] 2.2 Test simulation with null dates
  - Create duplicated campaign
  - Navigate to Simulation section
  - Verify it shows "Not Started" (not "Completed")
  - Verify all buttons are enabled (Run, Commit, Rollback, Rebalancing)

- [x] 2.3 Test module detail loading
  - Create duplicated campaign
  - Edit module dates
  - Load module detail/bidding details
  - Verify no 404 error

- [x] 2.4 Regression testing
  - Test existing campaigns still work
  - Test activatePhase functionality is not affected
  - Verify campaign list status strictly matches module detail constraints

## 3. Documentation

- [x] 3.1 Update API documentation if needed
  - Review NelmioApiDocBundle annotations in modified controllers

- [x] 3.2 Document the changes
  - Explain the null date handling fix
  - Explain the FK relationship fix
  - Explain the isActive flag fix
  - Explain the is_active string phase logic mapping
  - Explain the campaign list module status aggregation fix
