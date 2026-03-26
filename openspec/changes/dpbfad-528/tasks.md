## 1. Backend — Fix Credit Progress Calculation

- [x] 1.1 In `bidding-api/src/Domain/Dashboard/PMDashboardStatsService.php`, update `getCreditProgress()` to read `min_credits_per_student` / `max_credits_per_student` from the Final Enrollment module's own phase config first. Only fall back to the bidding round config if the Final Enrollment config does not define these values. Currently the code at ~line 240 only reads from the bidding round config (`$brConfig`).
- [x] 1.2 Verify the `getBiddingStatus()` method in `PMDashboardStatsService.php` correctly aggregates across all Pre-Bidding and Bidding rounds. Review `CampaignStatsBatchService::collectBiddingPhases()` to confirm it collects both `pre_bidding` and `bidding_round` modules. No changes expected — just verify and document findings.
- [x] 1.3 Update the empty stats label in `getEmptyStats()` and the empty-return paths in `getBiddingStatus()` and `getCreditProgress()` to ensure label text is consistent with the spec format: `"{percentage}% students have done bidding (Totals across all active campaigns' Pre-Bidding and Bidding rounds)"` and `"{percentage}% students are on track (Totals across all active campaigns' Final Enrollment)"`. (Already looks correct — verify no discrepancy.)

## 2. Testing

- [x] 2.1 Add a unit test for `PMDashboardStatsService::getCreditProgress()` in `bidding-api/tests/Unit/Domain/Dashboard/` covering: (a) Final Enrollment with its own credit config, (b) Final Enrollment without credit config falling back to bidding round, (c) no active campaigns with Final Enrollment returns 0/0, (d) multiple campaigns aggregation.
- [x] 2.2 Add a unit test for `PMDashboardStatsService::getBiddingStatus()` covering: (a) multiple campaigns with multiple Pre-Bidding + Bidding rounds, (b) no active campaigns returns 0/0.

## 3. Verification

- [ ] 3.1 Manual smoke test: Log in as PM, check Bidding Status Overview shows correct numbers across active campaigns.
- [ ] 3.2 Manual smoke test: Log in as PM, check Credit Progress shows corrected numbers using Final Enrollment credit config ranges.
- [ ] 3.3 Manual smoke test: Log in as PO, verify the same metrics are displayed (PO shares PM API endpoint).
