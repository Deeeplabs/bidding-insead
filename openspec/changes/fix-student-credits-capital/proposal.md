## Why

After the initial round of fixes for negative values and hardcoded placeholders, **three categories of bugs persist** in the student credits and capital system:

### Bug Category 1: PM Dashboard Aggregate Stats Not Updating

The PM Dashboard "Overall Dashboard Stats" has three cards:
1. **Active Campaigns** → Works correctly (shows 20).
2. **Bidding Status Overview** → Shows `0 / 15` even after creating a new campaign with an active bidding round containing 5 eligible students.
3. **Credit Progress** → Shows `0 / 0` regardless.

**Root Cause**: 
- **Bidding Status**: `PMDashboardStatsService::getBiddingStatus()` uses strict date-range checks (`$now >= $startDate && $now <= $endDate`) to identify active bidding phases. If the phase configuration's `isActive` flag is `false` despite having valid dates (common with duplicated campaigns), the phase is skipped entirely at line 113 (`if (!$pc->isActive()) continue`). Additionally, the `findTotalBidCompletedByCampaign` query may not find "completed" students because no bids have been submitted yet — but the method should still correctly count the eligible total.
- **Credit Progress**: `getCreditProgress()` only counts students during `final_enrollment` phases. Since the new campaign is still in the `bidding_round` phase, there are zero active `final_enrollment` phases, resulting in `0 / 0`. This is actually **by design** — credit progress doesn't apply until final enrollment. But the UI label is misleading.

### Bug Category 2: Student Credits/Capital Not Reflecting New Campaign

When a new campaign (e.g., "FISH BLOOB…") is created with:
- Capital Granted: 900 (campaign-level)
- Min/Max Credits: 10–70
- 5 eligible students

**The student list and student dashboard do NOT adjust credits or capital** because:

1. **`getInitialCapital()` (single-student path)** accumulates capital from campaigns the student **participated in** (determined by having `Bid` records). A brand-new campaign with no bids means this campaign's capital (900) is NOT included → student sees old accumulated capital only (e.g., 54 left for Dery).

2. **`getBatchCapital()` (list path)** uses completely different logic: it reads `PromotionSetting::getInitialCapital()` — a static per-promotion value that has NOTHING to do with campaign-configured capital. This causes:
   - Different capital values between the student list page and the student dashboard.
   - Capital not reflecting campaign-level grants at all.

3. **Bidding round view** (`CampaignToModuleDetailDtoMapper::buildBiddingRoundDetail`) uses `$campaign->getMinCapitalGranted()` as `granted` and computes `left = granted - spent` per-module. This shows **900** Total Capital on the bidding page, while the student dashboard shows **54** Capital Left — a complete mismatch.

4. **Credits**: `StudentCreditService::getTotalCreditGranted()` accumulates `minCreditsToFulfill` from campaigns via bids. No bids yet → new campaign's credits not included.

### Bug Category 3: Inconsistent Student List Count (5 vs 9)

The student list sometimes shows 5 results and sometimes 9 when refreshing. 

**Root Cause**: The `StudentRepository::queryPaginated()` joins `s.promotion` with `p_stat = ACTIVE`, but some students may belong to **multiple promotions** or have their promotion status toggling. More likely, it's a **Doctrine Paginator** issue with `fetchJoinCollection: true` combined with `leftJoin('s.studentData', 'sd')` — when `studentData` has multiple records or NULL, the paginator's count query and result query can diverge.

## What Changes

### Phase 1: Fix Capital Source Consistency
- **Unify `getInitialCapital()` and `getBatchCapital()`** to both use campaign-level capital (`getMinCapitalGranted()`) from campaigns the student is **eligible for** (not just participated in via bids). Include campaigns where the student is in the eligible student list, even without bids.
- **Add campaign eligibility lookup**: When a student is eligible for a campaign (per `student_selection` config), that campaign's capital grant should be included in their capital calculation, regardless of whether they've placed bids yet.
- **Apply `max(0, ...)` in `getBatchCapital()`** — currently missing (line 225: `left: $capital - $spent` without the floor).

### Phase 2: Fix Credits Source for New Campaigns
- **Extend `getTotalCreditGranted()`** to include campaigns the student is eligible for, not just those with existing bids.
- **Extend `getBatchTotalCredits()`** for consistency with the single-student path.

### Phase 3: Fix PM Dashboard Stats
- **Bidding Status**: Ensure `getBiddingStatus()` counts eligible students even for campaigns where no bids have been submitted. The issue is that the Tier 1 date check is correct but `findTotalBidCompletedByCampaign` returns 0 (expected when no one has bid). The "total" denominator should still reflect all eligible students across active bidding rounds.
- **Credit Progress label**: Clarify that credit progress only applies to final enrollment phases.

### Phase 4: Fix Student List Consistency
- **Investigate and fix the Doctrine Paginator** issue causing inconsistent counts. May need to switch from `fetchJoinCollection: true` to a count query approach, or ensure the join doesn't produce duplicates.

## Capabilities

### Modified Capabilities
- `student-dashboard-stats`: Fix capital/credit source to include campaign eligibility (not just bid participation)
- `pm-dashboard-stats`: Ensure bidding status denominator reflects all eligible students
- `student-list`: Fix capital calculation consistency between batch and single paths; fix pagination inconsistency

## Impact

### bidding-api (Backend)
- **StudentCapitalService.php**: 
  - `getInitialCapital()`: Change from bid-based campaign lookup to eligibility-based campaign lookup
  - `getBatchCapital()`: Replace PromotionSetting-based initial capital with campaign-level capital matching `getInitialCapital()` logic; add `max(0, ...)` floor
- **StudentCreditService.php**:
  - `getTotalCreditGranted()`: Include campaigns student is eligible for (not just bid-participated)
  - `getBatchTotalCredits()`: Align with single-student path
- **PMDashboardStatsService.php**:
  - `getBiddingStatus()`: Verify that eligible total is populated even when no bids exist
  - `getCreditProgress()`: Review label clarity
- **CampaignToModuleDetailDtoMapper.php**:
  - `buildBiddingRoundDetail()`: Already uses `$campaign->getMinCapitalGranted()` — no change needed, but document the intended calculation (per-campaign capital)
- **StudentRepository.php**:
  - `queryPaginated()`: Investigate Doctrine Paginator inconsistency

### bidding-web (Student Portal)
- **No changes required** — uses existing API response fields

### Database
- No changes expected

### Migration Risk
- Medium — capital calculation logic change affects all capital consumers
- The switch from PromotionSetting-based to campaign-based capital in `getBatchCapital()` is a semantic change
- Backward compatibility maintained: API response shape unchanged

## Scenario Validation

### Test Case: New Campaign (Image 2 Scenario)
1. Campaign "FISH BLOOB…" created with Capital=900, Credits=10–70
2. 5 eligible students selected
3. **Before fix**: Student list shows old capital values (40, 13.5, 0, 25, 52 credits / 54, 0, 0, 0, 140 capital)
4. **After fix**: Student list should reflect new campaign's 900 capital grant added, credits should include the new campaign's min credits target

### Test Case: PM Dashboard (Image 1 Scenario)
1. 20 active campaigns, newly created campaign with bidding round active
2. **Before fix**: Bidding Status = 0/15, Credit Progress = 0/0
3. **After fix**: Bidding Status total should include eligible students from new bidding round; Credit Progress = 0/0 is correct (no final enrollment active)

### Test Case: Student Dashboard vs Bidding Round (Image 3 Scenario)
1. Student Dery sees Capital Left = 54 on dashboard
2. Bidding round shows Total Capital = 900
3. **Before fix**: Different capital sources → mismatch
4. **After fix**: Dashboard capital_left should include the new campaign's 900 grant; bidding page shows per-campaign view (900)
