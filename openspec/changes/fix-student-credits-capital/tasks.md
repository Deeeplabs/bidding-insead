# Implementation Tasks — Fix Student Credits & Capital (Non-Negative Values)

## 1. Backend — Fix Capital and Credits Calculations

- [x] 1.1 Fix `StudentCapitalService::getCapital()` — add `max(0, $capital - $spent)` guard at line 44 in `bidding-api/src/Domain/Student/StudentCapitalService.php`
- [x] 1.2 Fix `CampaignToActiveBiddingRoundDtoMapper::buildBiddingRoundModuleData()` — add `max(0, $capitalGranted - $cumulativePoints)` guard at line 402 in `bidding-api/src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapper.php`
- [x] 1.3 Fix `StudentCreditService::getCredits()` — replace hardcoded `left: 1` with `max(0, $totalCredits->getTotal() - $totalCreditsEarned)` at line 60 in `bidding-api/src/Domain/Student/StudentCreditService.php`
- [x] 1.4 Fix `StudentStatsService` — update `credits_to_be_fulfilled` calculation to use `max(0, $creditsGranted - $creditsEarned)` in `bidding-api/src/Domain/Student/Dashboard/StudentStatsService.php`
- [x] 1.5 Fix PM Student List Match — Refactor `StudentListToDtoMapper`, `StudentToDtoMapper`, `StudentDataToDtoMapper`, and `StudentService::listStudents` mapping to use the dynamically computed `credits_to_be_fulfilled` (`Credits` column) and `capital->getLeft()` (`Capital Left` column).

## 2. Manual Verification

- [x] 2.1 Test dashboard stats endpoint (`GET /student/dashboard/stats`) — verify capital_left is >= 0
- [x] 2.2 Test student capital endpoint (`GET /student/capital`) — verify capital left is >= 0
- [x] 2.3 Test active bidding round endpoint — verify module capital_left is >= 0
- [x] 2.4 Test with student having single campaign — verify capital_left and credits calculations
- [x] 2.5 Test with student having multiple campaigns — verify accumulation works correctly
- [x] 2.6 Test with manual capital adjustment — verify adjustment is included in capital_left
- [x] 2.7 Test with student who has overspent capital — verify capital_left shows 0, not negative
- [x] 2.8 Test with student who has over-fulfilled credits — verify credits_to_be_fulfilled shows 0, not negative
- [x] 2.9 Test backward compatibility — existing API consumers work without changes