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

## 2. Testing Tasks

- [ ] 2.1 Test campaign duplication flow
  - Create a new campaign by duplicating an existing one
  - Verify new campaign modules have proper FK relationships
  - Verify no null campaign_module_id in database

- [ ] 2.2 Test simulation with null dates
  - Create duplicated campaign
  - Navigate to Simulation section
  - Verify it shows "Not Started" (not "Completed")
  - Verify all buttons are enabled (Run, Commit, Rollback, Rebalancing)

- [ ] 2.3 Test module detail loading
  - Create duplicated campaign
  - Edit module dates
  - Load module detail/bidding details
  - Verify no 404 error

- [ ] 2.4 Regression testing
  - Test existing campaigns still work
  - Test activatePhase functionality is not affected

## 3. Documentation

- [ ] 3.1 Update API documentation if needed
  - Review NelmioApiDocBundle annotations in modified controllers

- [ ] 3.2 Document the changes
  - Explain the null date handling fix
  - Explain the FK relationship fix
  - Explain the isActive flag fix
