# Implementation Tasks - Student Dashboard Stats (Backend Fix Only)

## 1. Backend - Fix Capital and Credits Calculation

- [x] 1.1 Review existing StudentStatsController.php
- [x] 1.2 Review existing StudentStatsDto.php (no changes needed - existing fields used)
- [x] 1.3 Review existing StudentStatsService.php
- [x] 1.4 Modify StudentCapitalService to calculate capital from Campaign.minCapitalGranted via Bid entity
- [x] 1.5 Modify StudentCreditService to calculate credits_to_be_fulfilled from Campaign.minCreditsToFulfill via Bid entity
- [x] 1.6 Add BidRepository to StudentCapitalService
- [ ] 1.7 Test API endpoint manually with curl to verify response

## 2. Manual Verification

- [ ] 2.1 Test with student having single campaign - verify capital_left calculation
- [ ] 2.2 Test with student having multiple campaigns - verify accumulation
- [ ] 2.3 Test with manual capital adjustment - verify adjustment is included
- [ ] 2.4 Test during pre-bidding phase - verify spent fields are 0
- [ ] 2.5 Test after simulation - verify credits_to_be_fulfilled calculation
- [ ] 2.6 Test backward compatibility - existing API consumers work without changes

## 3. Deployment

- [ ] 3.1 Deploy to DEV environment
- [ ] 3.2 Run smoke tests in DEV
- [ ] 3.3 Deploy to INT environment
- [ ] 3.4 Deploy to UAT for PM validation
- [ ] 3.5 Deploy to PROD
