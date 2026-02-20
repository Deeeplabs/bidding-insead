## 1. Make DTO Fields Nullable (List Endpoint Only)

- [x] 1.1 **Update `CampaignListDto` field types** 
  - File: `bidding-api/src/Domain/Campaign/Campaign/CampaignListDto.php`
  - ✅ Already done — `?int` with `nullable: true` in OA attributes

## 2. Rewrite calculateCampaignStatistics() with 3-Tier Logic

- [x] 2.1 **Rewrite `calculateCampaignStatistics()` main body**
  - File: `bidding-api/src/Domain/Campaign/Campaign/CampaignService.php`
  - Replace the entire method body with 3-tier approach:
    1. Single iteration through executions → phaseConfigs:
       - Any phase with `start_date <= now <= end_date` → `$hasActivePhase = true`
       - Pre_bidding/bidding_round phases with active dates → collect into `$activeBiddingPhases[]`
    2. **Tier 3**: If `!$hasActivePhase` → return null
    3. **Tier 1**: If `$activeBiddingPhases` not empty → call `aggregateBiddingStats()` helper
    4. **Tier 2**: If `$activeBiddingPhases` empty (non-bidding module active) → use `getCurrentActivePhaseDetailsByModule('bidding_round', true)` to get closest past bidding round → call `calculateStatsFromPhase()` helper
    5. Final fallback: return 0/0/0 if no past bidding round found

- [x] 2.2 **Add `aggregateBiddingStats()` helper method**
  - File: `bidding-api/src/Domain/Campaign/Campaign/CampaignService.php`
  - Private method that loops through collected `$activeBiddingPhases` and sums:
    - Eligible students per phase (`getEligibleStudents()`) — with correct config including student_filters
    - Completed bids per phase (pre_bidding: `findCompletedSubmissionByCampaign()` → fallback `findTotalBidCompletedByCampaign()`; bidding_round: `findTotalBidCompletedByCampaign()`)
  - Returns aggregated `['total_students' => X, 'bidding_pending_count' => Y, 'bidding_completed_count' => Z]`

- [x] 2.3 **Add `calculateStatsFromPhase()` helper method**
  - File: `bidding-api/src/Domain/Campaign/Campaign/CampaignService.php`
  - Private method that calculates stats from a single phase (used for Tier 2 / closest past bidding round):
    - Gets eligible students from phase config — with correct config including student_filters
    - Gets completed bids from `findTotalBidCompletedByCampaign()`
    - Returns stats array

## 3. Verification

- [x] 3.1 **Verify Tier 1: Both pre-bidding AND bidding round active → aggregated stats**
  - Campaign with two module groups both currently active
  - Expected: summed total_students and bidding counts
  - Verify total matches preview page: (pre-bidding eligible) + (bidding round eligible)

- [x] 3.2 **Verify Tier 2: Bidding completed, add/drop active → past bidding round stats**
  - Campaign like "Playwright | 19 Feb 2026 | 1IHTIR" where bidding is complete but add/drop is active
  - Expected: stats from the completed bidding round (NOT 0/0/0)

- [x] 3.3 **Verify Tier 3: All completed → null stats**
  - Campaign with all modules completed
  - Expected: null for all three stats

- [x] 3.4 **Verify Tier 3: Draft / future → null stats**  
  - Expected: null for all three stats

- [x] 3.5 **Verify student_filters are applied correctly in campaign list stats**
  - Campaign with student_include/student_exclude filters configured
  - Compare campaign list `total_students` with preview page "Eligible Students" count
  - Pre-Bidding eligible + Bidding Round eligible should equal campaign list total_students
  - Example: preview shows 469 per module → list should show 938 (469+469)
