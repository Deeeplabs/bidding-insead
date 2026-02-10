## 1. Fix CampaignPhaseConfig isActive During Duplication

- [x] 1.1 In `bidding-api/src/Domain/Campaign/Campaign/CampaignDuplicationService.php`, change the CampaignPhaseConfig cloning block (around line 280) to set `$newPhaseConfig->setIsActive(false)` instead of `$newPhaseConfig->setIsActive($sourcePhaseConfig->isActive())`

## 2. Add Draft Status Guard to Admin Module Status Computation

- [x] 2.1 In `bidding-api/src/Domain/Campaign/Campaign/CampaignService.php`, identify how to check if a campaign is in Draft status (look up `Campaign::getStatus()` — returns string `'draft'`)
- [x] 2.2 In `CampaignService.php` `$buildModuleData` closure (around lines 390-470), add an early check: if the campaign status is Draft, set `$is_active = 'CLOSE'` for all modules and skip date-range evaluation. Also set `$isCurrentlyActive = false` for all phase configs.

## 3. Add Draft Status Guard to Student Module Status Mapper

- [x] 3.1 In `bidding-api/src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToModuleDetailDtoMapper.php` (around line 341), add a check: if the campaign is in Draft status, set `$moduleItem->is_active = false` and skip the date-range computation

## 4. Add Draft Status Guard to ActiveBiddingRound Mapper

- [x] 4.1 In `bidding-api/src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapper.php` (around line 840), add the same Draft status guard to prevent `$moduleItem->is_active = true` for Draft campaigns

## 5. Auto-transition Campaign from Draft to Open on Module Activation

- [x] 5.1 In `bidding-api/src/Domain/Campaign/Campaign/CampaignPhaseService.php` `activatePhase()` method, after the phase `startDate` is set (when `$status === 'open'`) and before `$this->entityManager->flush()`, add a check: if the campaign is in `draft` status, set `$campaign->setStatus('open')` and persist the campaign. Navigate from phaseConfig → campaignExecution → campaignModule → campaign.
- [x] 5.2 Add PHPUnit test for `CampaignPhaseService::activatePhase()`: verify that when opening a phase on a draft campaign, the campaign status transitions to `open`
- [x] 5.3 Add PHPUnit test for `CampaignPhaseService::activatePhase()`: verify that when opening a phase on an already-open campaign, the campaign status remains `open`
- [x] 5.4 Add PHPUnit test for `CampaignPhaseService::activatePhase()`: verify that closing a phase does NOT change the campaign status

## 6. Unit Tests for Existing Fixes

- [x] 6.1 Add PHPUnit test in `bidding-api/tests/Unit/Domain/Campaign/` for `CampaignDuplicationService`: verify that duplicated `CampaignPhaseConfig` records have `isActive = false`
- [x] 6.2 Add PHPUnit test for `CampaignService` module status computation: verify that Draft campaigns return `is_active = 'CLOSE'` for all modules regardless of date ranges
