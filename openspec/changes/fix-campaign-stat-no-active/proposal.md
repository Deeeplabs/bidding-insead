## Why

Campaign list stats (`total_students`, `bidding_completed_count`, `bidding_pending_count`) have three problems:

### Problem 1: Stats show 0/0/0 when downstream module is active after bidding completes

When bidding round is completed but a downstream module (simulation, add/drop & waitlist) is still active, the stats should be calculated from the **completed bidding round data**. Currently returns 0/0/0 because the code only looks at truly active (`start_date <= now <= end_date`) pre_bidding/bidding_round phases and finds none.

**Example (from screenshot):**
- Bidding Round: completed ✅
- Simulation: completed ✅
- Add/Drop & Waitlist: ACTIVE 🟢→
- Stats show: 0/0/0 ← WRONG, should show bidding round stats (e.g., 33/14/19)

### Problem 2: Stats not aggregated across multiple active pre-bidding/bidding modules

When a campaign has both Pre-Bidding (33 students) AND Bidding Round (33 students) actively running, stats should aggregate to 66 total. Currently only picks ONE phase.

### Problem 3: Completed campaigns show stale stats instead of null

`getCurrentActivePhaseDetails()` has a `final_enrollment` fallback that returns a module even when all phases have ended, so the null-stat check is always bypassed.

### Root Cause

The `calculateCampaignStatistics()` method needs a **3-tier strategy**:

1. **Active pre_bidding/bidding_round phases exist** → aggregate stats across all active phases
2. **No active pre_bidding/bidding_round, but other module (simulation/add_drop) is active** → fall back to the closest past bidding round for stats
3. **No phase of any type is active** → return null

Currently the code only handles tiers 1 and 3, missing tier 2 entirely (returning 0/0/0 instead of past bidding round stats).

## What Changes

Rewrite `calculateCampaignStatistics()` to implement the 3-tier strategy:

1. **Tier 1** (active pre_bidding/bidding_round): Aggregate stats across all active phases
2. **Tier 2** (non-bidding module active, bidding completed): Use `getCurrentActivePhaseDetailsByModule('bidding_round', true)` to get the closest past bidding round, then calculate stats from that
3. **Tier 3** (nothing active): Return null

## Capabilities

### Modified Capabilities
- `campaign-statistics`: 3-tier strategy for stats calculation

## Impact

**Affected code (bidding-api only):**
- `src/Domain/Campaign/Campaign/CampaignService.php` — `calculateCampaignStatistics()` method
- `src/Domain/Campaign/Campaign/CampaignListDto.php` — already nullable

**Not affected:**
- `src/Entity/Campaign.php` — NOT modified
- No database migration required
