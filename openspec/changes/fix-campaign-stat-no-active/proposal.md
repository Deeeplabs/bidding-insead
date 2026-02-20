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

### Problem 4: Total students count must respect student_filters (include/exclude)

The `total_students` on the campaign list must match what the preview page shows. Both code paths use `getEligibleStudents()` which accounts for:
- **DB table filters** (`campaign_student_filter` entity, filtered by `$phaseConfigId`)
- **Config-level filters** (`$config['student_filters']` from `moduleConfig.config`)
- **Student selection** (`$config['student_selection']` — `include_all_students`, `include_filtered_students_only`)
- **Include/exclude students** (`student_include` / `student_exclude` filter types in both DB and config filters)

The config object passed to `getEligibleStudents()` is built identically in both our Tier 1 code and the entity's `getCurrentActivePhaseDetailsByModule()` (used by Tier 2):
```
$moduleConfig = $phaseConfig->getModuleConfig();
'config' => array_merge(
    $moduleConfig['config'] ?? [],             // includes student_filters, selected_student_ids
    isset($moduleConfig['student_selection'])
        ? ['student_selection' => $moduleConfig['student_selection']]
        : []
),
```

The preview page uses a **different endpoint** (`GET /students` with `mapStudentFiltersToQuery`) which applies include/exclude as SQL query params. Both should return equivalent results, but the canonical source for bidding logic is `getEligibleStudents()`.

### Root Cause

The `calculateCampaignStatistics()` method needs a **3-tier strategy**:

1. **Active pre_bidding/bidding_round phases exist** → aggregate stats across all active phases
2. **No active pre_bidding/bidding_round, but other module (simulation/add_drop) is active** → fall back to the closest past bidding round for stats
3. **No phase of any type is active** → return null

Currently the code only handles tiers 1 and 3, missing tier 2 entirely (returning 0/0/0 instead of past bidding round stats).

## What Changes

Rewrite `calculateCampaignStatistics()` to implement the 3-tier strategy:

1. **Tier 1** (active pre_bidding/bidding_round): Aggregate stats across all active phases using `getEligibleStudents()` which correctly applies student_filters (include/exclude)
2. **Tier 2** (non-bidding module active, bidding completed): Use `getCurrentActivePhaseDetailsByModule('bidding_round', true)` to get the closest past bidding round, then calculate stats from that using the same `getEligibleStudents()` call
3. **Tier 3** (nothing active): Return null

Both Tier 1 and Tier 2 pass the correct `config` (including `student_filters`, `student_selection`, `selected_student_ids`) and `phaseConfigId` to `getEligibleStudents()`, ensuring include/exclude logic is applied consistently.

## Capabilities

### Modified Capabilities
- `campaign-statistics`: 3-tier strategy for stats calculation with proper student_filters handling

## Impact

**Affected code (bidding-api only):**
- `src/Domain/Campaign/Campaign/CampaignService.php` — `calculateCampaignStatistics()` method
- `src/Domain/Campaign/Campaign/CampaignListDto.php` — already nullable

**Not affected:**
- `src/Entity/Campaign.php` — NOT modified
- `src/Domain/Campaign/ActiveCampaign/CampaignStudentEligibilityService.php` — NOT modified (used as-is)
- No database migration required
