# Fix Student Dashboard Credits and Capital — PM Rule Alignment (Phase 2)

## Problem

After initial fixes for negative values, profound architectural bugs remained in how credits and capital were sourced and displayed across the application, especially when campaigns were newly created. The root causes spanned multiple services and query logic:

1. **Capital Source Inconsistency (3 divergent paths)**:
   - The Student Dashboard computed capital by summing grants from campaigns the student **had bids in**, ignoring newly eligible campaigns where the student hadn't bid yet.
   - The Student List page mapped initial capital straight from `PromotionSetting`, an unrelated legacy table giving inconsistent and wrong values.
   - The Bidding Module View correctly used the campaign-level grant.
2. **Credits Source Inconsistency**: Similar to capital, the Student Dashboard sourced credit targets (`min_credits_to_fulfill`) only from bid-participated campaigns rather than eligible campaigns.
3. **PM Dashboard Bidding Status Divergence**: The "Bidding Status Overview" widget showed `0 / n` (e.g., `0 / 15` instead of `0 / 20`) for newly created campaigns because duplicating a campaign often sets the underlying `phase-config` `isActive` constraint to false despite valid date ranges. It artificially restricted eligible denominator counts.
4. **Student List Total Count Oscillation**: Running refresh on the student list could randomly return 5 vs 9 results. Paginator logic applied `fetchJoinCollection: true` on queries with a `LEFT JOIN` to `studentData`.

## Solution

Reworked core logic across capital and credit services to ensure that definitions rely on **Campaign Eligibility** rather than Bid participation. Synced Doctrine paginator behavior.

### Changes Made

**Modified Files:**

1. **`src/Domain/Student/StudentCapitalService.php`**
   - Implemented `getStudentEligibleCampaigns()` unifying bid-participated campaigns + campaigns explicitly including a student via SQL `CampaignStudentFilter` and JSON configs.
   - Rewrote `getInitialCapital()` and `getBatchCapital()` to source exclusively from `campaign->getMinCapitalGranted()` of all eligible campaigns.
   - Applied the missing `max(0, $capital - $spent)` clamp natively into `getBatchCapital()`.

2. **`src/Domain/Student/StudentCreditService.php`**
   - Transferred source query context for `getTotalCreditGranted()` to ingest results via the newly implemented `getStudentEligibleCampaigns()` model, surfacing the credit targets the moment a student is considered eligible rather than conditionally post-bidding.
   - Rewrote `getBatchTotalCredits()` batch mapping to directly lean on unified query logic for safety.

3. **`src/Domain/Dashboard/PMDashboardStatsService.php`**
   - Removed strict `isActive` boolean gating inside `getBiddingStatus()` and `getCreditProgress()`. Relies natively on the valid start/end scheduling boundary (`$now >= $startDate && $now <= $endDate`). Forces active phases inside duplicated campaigns to naturally count all eligible students in the aggregate denominator.

4. **`src/Repository/StudentRepository.php`**
   - Forced `queryPaginated()` Doctrine Paginator setup into `fetchJoinCollection: false` resolving the random counting offset when evaluating arrays with `LEFT JOIN studentData`.
   - Injected `->select('DISTINCT s')` directly into the ORM QueryBuilder.

## Impact

- **Data Accuracy:** `Capital Left` natively syncs between the Student Dashboard and Student Lists immediately for eligible participants on brand new campaigns.
- **Data Accuracy:** `Credits to be fulfilled` target correctly incorporates requirements the moment student matching is performed on campaign publishing.
- **Data Consistency:** Eliminates the student list total result fluctuation glitch.
- **Data Accuracy:** `Bidding Status Overview` PM widget actively represents true student-reach totals regardless of duplicated active flags.
- **Compatibility:** Fully backwards consistent. No DTO definition/type transformations.

## Testing

Verified through code review:
- [x] Scenario ST 1: PM Dashboard denominator appropriately sizes active participants.
- [x] Scenario ST 1: New Campaigns securely distribute `Capital Granted` onto students without requiring a pre-existing bid trigger.
- [x] Scenario ST 1: New Campaigns distribute `Credits to be fulfilled` accurately across batch and localized services.
- [x] Scenario ST 1: Student List pagination outputs consistent counts predictably.
- [x] Backend-only changes — no frontend modifications required.
