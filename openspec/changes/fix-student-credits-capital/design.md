## Context

After the first round of fixes (negative values, hardcoded `left: 1`), three bug categories remain:

1. **Capital/credit source mismatch**: The single-student path (`getCapital()` / `getInitialCapital()`) accumulates capital from campaigns the student **has bids in**. The batch path (`getBatchCapital()`) uses **PromotionSetting initial capital** — a completely different data source. Neither path includes capital from campaigns the student is **eligible for but hasn't bid in yet**.

2. **PM Dashboard stats stale**: `PMDashboardStatsService::getBiddingStatus()` correctly counts active bidding phases but the "completed" count is 0 because no bids have been submitted. The "total" denominator should still update. `getCreditProgress()` shows 0/0 because it only counts `final_enrollment` phases (by design).

3. **Student list count instability**: `StudentRepository::queryPaginated()` with `fetchJoinCollection: true` and a `leftJoin` on `studentData` can produce inconsistent counts due to Doctrine Paginator's count vs result query divergence.

## Goals / Non-Goals

**Goals:**
- Ensure capital calculations use **campaign-level capital grants** (`getMinCapitalGranted()`) consistently across all code paths
- Include a student's **eligible campaigns** (not just bid-participated) when computing capital and credit totals
- Fix `getBatchCapital()` to align with `getCapital()` logic (campaign-based instead of PromotionSetting-based)
- Add missing `max(0, ...)` floor in `getBatchCapital()`
- Fix or investigate the student list count inconsistency
- Ensure PM Dashboard Bidding Status total reflects all eligible students from active bidding rounds

**Non-Goals:**
- No changes to frontend (bidding-web)
- No changes to DTO field names or types
- No new API endpoints
- No database schema changes
- Credit Progress showing 0/0 when no final_enrollment is active is by design — no fix needed

## Decisions

### 1. Capital Source — Campaign-Level vs Promotion-Level
**Decision**: Both `getInitialCapital()` and `getBatchCapital()` should compute initial capital by summing `campaign->getMinCapitalGranted()` from all campaigns the student is **eligible for** (not just ones they have bids in).

**Rationale**: 
- The PM sets capital at the **campaign level** (`min_capital_granted` field on Campaign entity). This is the authoritative source.
- Using `PromotionSetting::getInitialCapital()` is wrong — that's a legacy per-promotion value unrelated to campaign bidding capital.
- Using bid participation is too restrictive — a student should see their capital allocation as soon as they're eligible for a campaign, not only after placing their first bid.

**Implementation**: Create a new method `getStudentEligibleCampaigns(Student $student)` that finds campaigns where the student appears in the eligible student list (via `CampaignStudentEligibilityService` or campaign module config's `student_selection`). Use this in both `getInitialCapital()` and `getBatchCapital()`.

**Trade-off**: This requires querying campaign eligibility which may be slower than the current bid-lookup approach. Mitigate with batch operations and caching.

### 2. Batch Capital — Rewrite to Match Single Path
**Decision**: Rewrite `getBatchCapital()` to use campaign-level capital grants instead of `PromotionSetting`. Additionally, add the missing `max(0, ...)` floor.

**Rationale**: `getBatchCapital()` line 225 currently does `left: $capital - $spent` without the `max(0, ...)` guard that exists in `getCapital()` line 53. This inconsistency means the student list page can show negative capital while the student dashboard shows 0.

### 3. Credit Source — Campaign Eligibility
**Decision**: Update `getTotalCreditGranted()` to include campaigns the student is eligible for, not just those with existing bids.

**Rationale**: Same reasoning as capital — the PM configures `min_credits_to_fulfill` at the campaign level. A student should see their credit target as soon as they're eligible.

### 4. PM Dashboard Bidding Status
**Decision**: The current `getBiddingStatus()` logic is mostly correct. The issue is that with a brand-new campaign, the "completed" count is 0 (correct — no one has bid) but the "total" should still reflect eligible students. Verify this path works correctly with the new campaign data.

**Rationale**: The screenshots show 0/15 which suggests the denominator IS being computed (15 is from other campaigns' existing data). The new campaign's 5 students are not being counted, likely because the phase config `isActive` flag may be false despite having valid dates.

### 5. Student List Count — Doctrine Paginator
**Decision**: Investigate and fix the `fetchJoinCollection` behavior. Consider adding `DISTINCT` to the query or switching the paginator configuration.

**Rationale**: The `leftJoin('s.studentData', 'sd')` combined with `fetchJoinCollection: true` can cause the paginator's `COUNT` query to return different results than the actual data query when `studentData` has NULL or multiple rows.

## Risks / Trade-offs

**[Risk] Performance of campaign eligibility lookup** → **Mitigation**: For `getBatchCapital()`, batch-query eligible campaigns rather than per-student lookups. Use campaign's `student_filter_ids` or similar pre-computed eligibility.

**[Risk] Capital semantics change** → **Mitigation**: This changes what "initial capital" means from "promotion-level grant" to "campaign-level grant sum". Verify that no other consumers rely on the PromotionSetting-based capital.

**[Risk] Backward compatibility** → **Mitigation**: API response shape unchanged. Only the computed values change to be correct.

**[Trade-off] Eligibility-based vs bid-based capital** → After this change, a student will see their total capital increase as soon as they become eligible for a new campaign, before placing any bids. This is the correct UX since the student needs to know their capital budget to place bids.
