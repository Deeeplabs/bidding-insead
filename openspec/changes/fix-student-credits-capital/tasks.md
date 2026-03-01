# Implementation Tasks — Fix Student Credits & Capital (Non-Negative Values)

## 1. Backend — Fix Capital and Credits Calculations

- [x] 1.1 Fix `StudentCapitalService::getCapital()` — add `max(0, $capital - $spent)` guard at line 44 in `bidding-api/src/Domain/Student/StudentCapitalService.php`
- [x] 1.2 Fix `CampaignToActiveBiddingRoundDtoMapper::buildBiddingRoundModuleData()` — add `max(0, $capitalGranted - $cumulativePoints)` guard at line 402 in `bidding-api/src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapper.php`
- [x] 1.3 Fix `StudentCreditService::getCredits()` — replace hardcoded `left: 1` with `max(0, $totalCredits->getTotal() - $totalCreditsEarned)` at line 60 in `bidding-api/src/Domain/Student/StudentCreditService.php`

## 2. Manual Verification

- [ ] 2.1 Test dashboard stats endpoint (`GET /student/dashboard/stats`) — verify capital_left is >= 0
- [ ] 2.2 Test student capital endpoint (`GET /student/capital`) — verify capital left is >= 0
- [ ] 2.3 Test active bidding round endpoint — verify module capital_left is >= 0
- [ ] 2.4 Test with student having single campaign — verify capital_left and credits calculations
- [ ] 2.5 Test with student having multiple campaigns — verify accumulation works correctly
- [ ] 2.6 Test with manual capital adjustment — verify adjustment is included in capital_left
- [ ] 2.7 Test with student who has overspent capital — verify capital_left shows 0, not negative
- [ ] 2.8 Test with student who has over-fulfilled credits — verify credits_to_be_fulfilled shows 0, not negative
- [ ] 2.9 Test backward compatibility — existing API consumers work without changes

## 3. Deployment

- [ ] 3.1 Deploy to DEV environment
- [ ] 3.2 Run smoke tests in DEV
- [ ] 3.3 Deploy to INT environment
- [ ] 3.4 Deploy to UAT for PM validation
- [ ] 3.5 Deploy to PROD
