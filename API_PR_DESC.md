# Fix: Cross-promotion student campaign visibility for per-module student includes

## Problem

When an admin adds a student from a different promotion to a campaign for "Pre-bidding ONLY" (via the module configuration UI), the student cannot see the campaign on their dashboard or bidding page. This prevents cross-promotion students who are explicitly added to specific campaign phases from accessing those campaigns.

## Root Cause

The system stores per-module student filters in **two separate locations** without cross-referencing them:

1. **`campaign_student_filters` DB table** — for campaign-level filters
2. **`campaign_phase_configs.module_config` JSON blob** — for per-module filters (e.g., "Pre-bidding ONLY")

The campaign discovery flow in `ActiveCampaignService::getActiveBiddingRounds` had **two problems**:

### Problem 1: Discovery Gap (PRIMARY — this PR)

The cross-promotion discovery method `findOpenCampaignsByStudentInclude` searched **only** the `campaign_student_filters` DB table. When admins configure per-module student includes via the module configuration UI, those filters are saved **exclusively** to `module_config.config.student_filters` JSON — never to the DB table. So the campaign was never discovered at all for cross-promotion students.

```
Student Dashboard Request
  → getActiveBiddingRounds()
    → Step 1: Query by promotion     → Student not in this promotion ❌
    → Step 2: findOpenCampaignsByStudentInclude (DB table) → Not in table ❌
    → Campaign never discovered. Student sees nothing.
```

### Problem 2: Eligibility Check (previously fixed)

Even if a campaign was discovered, `isStudentEligibleForCampaign` checked `tableFilters` and `configFilters` sequentially rather than merging them, causing students in phase-specific configs to be rejected if campaign-level filters existed. This was fixed in a prior commit by merging both filter sources.

### 3. Completed campaigns show stale stats instead of null
`getCurrentActivePhaseDetails()` has a `final_enrollment` fallback, so completed campaigns never return null.

### Fix 1: JSON-based Cross-Promotion Discovery (NEW)

Added a third campaign discovery path that searches `module_config` JSON for `student_include` entries:

1. **New repository method**: `CampaignPhaseConfigRepository::findOpenCampaignsByStudentIncludeInConfig(studentId)` — queries `campaign_phase_configs.module_config` JSON using LIKE matching for `student_include` entries containing the student ID.
2. **Third discovery step** in `getActiveBiddingRounds` — calls the new method after the existing table-based lookup, with de-duplication via `$existingCampaignIds`.

```
Student Dashboard Request
  → getActiveBiddingRounds()
    → Step 1: Query by promotion                              → ❌
    → Step 2: findOpenCampaignsByStudentInclude (DB table)     → ❌
    → Step 3: findOpenCampaignsByStudentIncludeInConfig (JSON) → ✅ Found!
    → isStudentEligibleForCampaign (merged filters)            → ✅ Eligible!
    → Campaign appears on dashboard ✅
```

### Fix 2: Filter Merge (EXISTING — prior commit)

`isStudentEligibleForCampaign` merges `tableFilters` and `configFilters` before validation:

```php
$allFilters = array_merge($tableFilters, $configFilters);
if (!empty($allFilters)) {
    return $this->checkStudentAgainstFilters($student, $allFilters, $campaignId);
}
```

## Files Changed

| File | Change |
|------|--------|
| `src/Repository/CampaignPhaseConfigRepository.php` | Added `findOpenCampaignsByStudentIncludeInConfig` — searches `module_config` JSON for `student_include` entries |
| `src/Domain/Campaign/ActiveCampaign/ActiveCampaignService.php` | Added `CampaignPhaseConfigRepository` dependency + third cross-promotion discovery block in `getActiveBiddingRounds` |
| `src/Domain/Campaign/ActiveCampaign/CampaignStudentEligibilityService.php` | *(Prior commit)* Merged `tableFilters` + `configFilters` in `isStudentEligibleForCampaign` |

## Discovery & Eligibility Flow (Updated)

```
getActiveBiddingRounds(student)
│
├─ Step 1: Promotion-based query (existing)
│   → Find campaigns matching student's promotion
│   → For each: isStudentEligibleForCampaign() → add to eligible list
│
├─ Step 2: Table-based cross-promotion (existing)
│   → findOpenCampaignsByStudentInclude() searches campaign_student_filters table
│   → For each: skip duplicates → isStudentEligibleForCampaign() → add to eligible list
│
├─ Step 3: JSON-based cross-promotion (NEW)
│   → findOpenCampaignsByStudentIncludeInConfig() searches module_config JSON
│   → For each: skip duplicates → isStudentEligibleForCampaign() → add to eligible list
│
└─ Sort + paginate → return to student dashboard
```

**Tier 2 — Bidding completed, add/drop active (past bidding round):**
```json
{ "total_students": 469, "bidding_completed_count": 14, "bidding_pending_count": 455 }
```

- **Fixes visibility**: Cross-promotion students added via per-module config can now see the campaign
- **Phase-aware**: Students added to "Pre-bidding ONLY" see the campaign during pre-bidding but NOT during bidding round (enforced by `isStudentEligibleForCampaign` checking the active phase's config)
- **Preserves existing behavior**: Same-promotion students and table-based cross-promotion still work identically
- **De-duplication**: Campaigns found via multiple discovery paths appear only once
- **No database migration required**

## Testing

Manual verification required:

- [ ] **2.1** Create campaign for promotion A → add student from promotion B to Pre-bidding module config → impersonate student B → verify campaign appears on dashboard
- [ ] **2.2** Advance to Bidding Round phase → verify campaign does NOT appear for student B
- [ ] **2.3** Verify same-promotion students still see campaigns normally
- [ ] **2.4** Verify campaign-level `student_include` filters (DB table) still work
- [ ] **2.5** Verify de-duplication: student found via multiple paths → campaign appears only once
