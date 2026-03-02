# Implementation Tasks — Fix Student Credits & Capital (Phase 2)

## 1. Backend — Fix Capital Source Consistency

- [x] 1.1 **Create `getStudentEligibleCampaigns(Student $student)` method** in `StudentCapitalService` — Find all active (open, not expired) campaigns where the student appears in the eligible student list. Use campaign's student selection config or query `CampaignStudentFilter` table. Returns `Campaign[]`.

- [x] 1.2 **Rewrite `getInitialCapital(Student $student)`** in `StudentCapitalService` — Change from bid-based campaign lookup (`getStudentParticipatedCampaigns`) to eligibility-based lookup (`getStudentEligibleCampaigns`). Sum `campaign->getMinCapitalGranted()` from eligible campaigns.

- [x] 1.3 **Rewrite `getBatchCapital(array $students)`** in `StudentCapitalService` — Replace PromotionSetting-based initial capital with campaign-level capital. For each student, compute initial capital as sum of `getMinCapitalGranted()` from their eligible campaigns. Add `max(0, ...)` floor to `left` calculation at line 225.

- [x] 1.4 **Fix `getBatchCapital` missing `max(0, ...)` floor** — Line 225 currently does `left: $capital - $spent`. Change to `left: max(0, $capital - $spent)` to match `getCapital()` line 53.

## 2. Backend — Fix Credits Source for New Campaigns

- [x] 2.1 **Rewrite `getTotalCreditGranted(Student $student)`** in `StudentCreditService` — Change from bid-based campaign lookup to eligibility-based lookup (matching the capital approach). Sum `campaign->getMinCreditsToFulfill()` from eligible campaigns.

- [x] 2.2 **Align `getBatchTotalCredits(array $studentIds)`** in `StudentCreditService` — Currently uses `findTotalBiddingCreditsByStudents` (bid-based). Align with the eligibility-based approach for consistency.

## 3. Backend — Fix PM Dashboard Stats

- [x] 3.1 **Debug `getBiddingStatus()` denominator** in `PMDashboardStatsService` — Verify that the newly created campaign's eligible students are being counted in `totalEligible`. Check: (a) campaign appears in `findOpenCampaigns`, (b) phase config is found via date check, (c) `getEligibleStudents()` returns correct students.

- [x] 3.2 **Check phase config `isActive` flag** — Duplicated campaigns may have `isActive = false` on their phase configs. The `getBiddingStatus()` at line 113 skips `!$pc->isActive()`. Verify the new campaign's phase configs have `isActive = true`.

## 4. Backend — Fix Student List Inconsistent Count

- [x] 4.1 **Investigate Doctrine Paginator inconsistency** — The `StudentRepository::queryPaginated()` uses `fetchJoinCollection: true` with a `leftJoin('s.studentData', 'sd')`. Test whether this causes count divergence. Consider: (a) adding `DISTINCT` on `s.id`, (b) removing unused `sd` join, or (c) setting `fetchJoinCollection: false` if no collection joins exist.

- [x] 4.2 **Add `DISTINCT` to student query** — If `leftJoin` causes duplicates, add `->select('DISTINCT s')` or `->distinct(true)` to the query builder.

## 5. Manual Verification

- [x] 5.1 Test student dashboard after new campaign creation — Verify capital_left includes new campaign's 900 grant
- [x] 5.2 Test student list Credits column — Verify it matches dashboard credits_to_be_fulfilled
- [x] 5.3 Test student list Capital column — Verify it matches dashboard capital_left
- [ ] 5.4 Test PM Dashboard Bidding Status — Verify total includes new campaign's eligible students
- [ ] 5.5 Test PM Dashboard after bidding round activation — Verify bidding status count updates
- [ ] 5.6 Test student list consistency — Refresh 10 times, verify same result count every time
- [ ] 5.7 Test bidding round vs dashboard capital — Verify bidding round shows per-campaign (900), dashboard shows accumulated total minus spent
- [ ] 5.8 Test backward compatibility — Existing students with old campaigns still show correct values