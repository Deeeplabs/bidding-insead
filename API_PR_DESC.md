# Fix campaign list stats: 3-tier calculation + aggregation + student_filters

## Problem

Four issues with `calculateCampaignStatistics()` in `CampaignService.php`:

### 1. Stats show 0/0/0 when downstream module is active after bidding completes
When bidding round is completed but add/drop & waitlist is still active, stats should show the completed bidding round's data (e.g., 33/14/19), not 0/0/0.

### 2. Stats not aggregated across multiple active pre-bidding/bidding modules
When both pre-bidding (33 students) AND bidding round (33 students) are active simultaneously, total should be 66 — not just 33 from the first phase found.

### 3. Completed campaigns show stale stats instead of null
`getCurrentActivePhaseDetails()` has a `final_enrollment` fallback, so completed campaigns never return null.

### 4. Total students must respect student_filters (include/exclude)
The `total_students` count must match what the preview page shows per module. Both paths use `getEligibleStudents()` with the correct config (including `student_filters`, `student_selection`, `selected_student_ids`) and `phaseConfigId` to scope DB-level filters.

## Solution: 3-Tier Strategy

Complete rewrite of `calculateCampaignStatistics()` with three tiers:

| Tier | Condition | Action |
|------|-----------|--------|
| **Tier 1** | Pre-bidding or bidding_round is active (`start <= now <= end`) | Aggregate stats across ALL active pre_bidding + bidding_round phases |
| **Tier 2** | No bidding active, but downstream module (simulation/add_drop) IS active | Fall back to closest past bidding round via `getCurrentActivePhaseDetailsByModule('bidding_round', true)` |
| **Tier 3** | No phase of any type is active | Return `null` (frontend shows "-") |

### Method structure:

```
calculateCampaignStatistics()     ← main: iteration + tier routing
├── aggregateBiddingStats()       ← Tier 1: sum across all active pre_bidding + bidding_round
└── calculateStatsFromPhase()     ← Tier 2: single phase stats (past bidding round)
```

### Student filters flow (both tiers):

```
moduleConfig = phaseConfig->getModuleConfig()
  → { "config": { "student_filters": [...], "selected_student_ids": [...] }, "student_selection": {...} }

config = array_merge(moduleConfig['config'], {student_selection: moduleConfig['student_selection']})
  → { "student_filters": [...], "student_selection": {...}, "selected_student_ids": [...] }

getEligibleStudents($campaign, $config, $phaseConfigId)
  → DB table filters (campaign_student_filter, scoped by phaseConfigId)
  → Config filters (student_filters: include/exclude)
  → Student selection (include_all_students, include_filtered_students_only)
  → Selected student IDs (explicit ID restriction)
```

## Files Changed

| File | Change |
|------|--------|
| `src/Domain/Campaign/Campaign/CampaignService.php` | Rewrite `calculateCampaignStatistics()` + add `aggregateBiddingStats()` + add `calculateStatsFromPhase()` |
| `src/Domain/Campaign/Campaign/CampaignListDto.php` | Already nullable (`?int`) from previous iteration |

### NOT Changed

| File | Reason |
|------|--------|
| `src/Entity/Campaign.php` | Entity methods unchanged — `final_enrollment` fallback is intentional for other consumers |
| `src/Domain/Campaign/ActiveCampaign/CampaignStudentEligibilityService.php` | Used as-is — already handles student_filters correctly |
| Database | No migration needed — stats calculated dynamically |

## API Response Examples

**Tier 1 — Both pre-bidding + bidding_round active (aggregated with filters):**
```json
{ "total_students": 938, "bidding_completed_count": 10, "bidding_pending_count": 928 }
```

**Tier 2 — Bidding completed, add/drop active (past bidding round):**
```json
{ "total_students": 469, "bidding_completed_count": 14, "bidding_pending_count": 455 }
```

**Tier 3 — All completed:**
```json
{ "total_students": null, "bidding_completed_count": null, "bidding_pending_count": null }
```
