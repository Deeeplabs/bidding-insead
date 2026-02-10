## 1. Fix CampaignPhaseConfig isActive During Duplication

- [x] 1.1 In `bidding-api/src/Domain/Campaign/Campaign/CampaignDuplicationService.php`, change the CampaignPhaseConfig cloning block (around line 280) to set `$newPhaseConfig->setIsActive(false)` instead of `$newPhaseConfig->setIsActive($sourcePhaseConfig->isActive())`

## 2. Add Draft Status Guard to Admin Module Status Computation

- [x] 2.1 In `bidding-api/src/Domain/Campaign/Campaign/CampaignService.php`, identify how to check if a campaign is in Draft status (look up `SessionStatus` entity or `Campaign::getStatus()` pattern used elsewhere in the codebase)
- [x] 2.2 In `CampaignService.php` `$buildModuleData` closure (around lines 390-470), add an early check: if the campaign status is Draft, set `$is_active = 'CLOSE'` for all modules and skip date-range evaluation. Also set `$isCurrentlyActive = false` for all phase configs.

## 3. Add Draft Status Guard to Student Module Status Mapper

- [x] 3.1 In `bidding-api/src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToModuleDetailDtoMapper.php` (around line 344), add a check: if the campaign is in Draft status, set `$moduleItem->is_active = false` and skip the date-range computation

## 4. Add Draft Status Guard to ActiveBiddingRound Mapper

- [x] 4.1 In `bidding-api/src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapper.php` (around line 844), add the same Draft status guard to prevent `$moduleItem->is_active = true` for Draft campaigns

## 5. Unit Tests

- [x] 5.1 Add PHPUnit test in `bidding-api/tests/Unit/Domain/Campaign/` for `CampaignDuplicationService`: verify that duplicated `CampaignPhaseConfig` records have `isActive = false`
- [x] 5.2 Add PHPUnit test for `CampaignService` module status computation: verify that Draft campaigns return `is_active = 'CLOSE'` for all modules regardless of date ranges
