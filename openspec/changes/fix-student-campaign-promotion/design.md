## Context

When an admin adds a student from a different promotion to a campaign for "Pre-bidding ONLY" participation, the student cannot see the campaign on their dashboard. 

**Full Architecture of Student Campaign Discovery:**

The `ActiveCampaignService::getActiveBiddingRounds` method discovers campaigns for a student via two paths:

1. **Promotion-based query** (line 80-138): Finds campaigns matching the student's promotion. Cross-promotion students are missed here.
2. **Cross-promotion lookup** (line 141-169): Calls `CampaignStudentFilterRepository::findOpenCampaignsByStudentInclude(studentId)` to find campaigns where the student is explicitly included via a `student_include` record in the `campaign_student_filters` table.

**Dual Storage System for Student Filters:**

Student include/exclude filters are stored in TWO separate locations:
- **`campaign_student_filters` table** (DB records): Created via `CampaignFilterService::createStudentFilter()` or `syncStudentFilters()`. Used for campaign-level filters. Has optional `phaseConfig` FK for phase-level filters.
- **`campaign_phase_configs.module_config` JSON blob**: Stored at path `module_config.config.student_filters[]`. Created when admin saves module configuration via `saveWithModules`/`updateWithModules`. Used for per-module filters.

**Root Cause — Discovery Gap:**

When the admin configures per-module students (e.g., adds students for "Pre-bidding ONLY" via the admin UI's module configuration), the `student_include` filters are saved ONLY to `module_config.config.student_filters[]` in the `CampaignPhaseConfig` JSON blob. They are **never** persisted to the `campaign_student_filters` table.

`findOpenCampaignsByStudentInclude` searches ONLY the `campaign_student_filters` table. So:
- Cross-promotion student added via per-module config → `student_include` in JSON only → **NOT FOUND** by `findOpenCampaignsByStudentInclude` → Campaign never discovered → Student can't see campaign.

**Why the previous fix was insufficient:**

The previous fix (merging `tableFilters` and `configFilters` in `isStudentEligibleForCampaign`) correctly handles eligibility AFTER a campaign is discovered. But the campaign is never discovered in the first place for cross-promotion students with JSON-only student includes.

## Goals / Non-Goals

**Goals:**
- Fix cross-promotion campaign discovery to also find campaigns where student is included via `module_config.config.student_filters` JSON
- Ensure students added per-module can see the campaign on their dashboard during the configured phase
- Maintain the existing eligibility check behavior (already fixed merge logic)

**Non-Goals:**
- Unify the dual storage system (DB table vs JSON) — that's a larger architectural change
- Modify how admins add students to campaigns (admin UI)
- Change campaign eligibility logic for same-promotion students

## Decisions

**Decision: Add JSON-based cross-promotion lookup in `ActiveCampaignService`**

Add a new query to search `CampaignPhaseConfig.module_config` JSON for `student_include` entries containing the student's ID. This supplements (not replaces) the existing `findOpenCampaignsByStudentInclude` table-based lookup.

**Approach A (Recommended): New repository method with JSON search**

Add `findOpenCampaignsByStudentIncludeInConfig(studentId)` to query the `campaign_phase_configs` table using MySQL JSON functions or LIKE matching on `module_config`:

```php
// In a new or existing repository
public function findOpenCampaignsByStudentIncludeInConfig(string $studentId): array
{
    // Search module_config JSON for student_include entries containing this student ID
    // module_config structure: {"config": {"student_filters": [{"filter_type": "student_include", "filter_value": [...]}]}}
    $qb = $this->createQueryBuilder('pc')
        ->select('DISTINCT IDENTITY(ce.campaign)')
        ->join('pc.execution', 'ce')
        ->join('ce.campaign', 'c')
        ->where('pc.isActive = :isActive')
        ->andWhere('pc.isDeleted = :isDeleted')
        ->andWhere('c.status = :status')
        ->andWhere('c.isDeleted = :campaignNotDeleted')
        ->andWhere('c.endDate >= :now')
        ->andWhere('pc.moduleConfig LIKE :studentIncludePattern')
        ->andWhere('pc.moduleConfig LIKE :studentIdPattern')
        ->setParameter('isActive', true)
        ->setParameter('isDeleted', false)
        ->setParameter('status', 'open')
        ->setParameter('campaignNotDeleted', false)
        ->setParameter('now', new \DateTime())
        ->setParameter('studentIncludePattern', '%student_include%')
        ->setParameter('studentIdPattern', '%' . $studentId . '%');

    $campaignIds = $qb->getQuery()->getSingleColumnResult();

    if (empty($campaignIds)) {
        return [];
    }

    return $this->getEntityManager()
        ->getRepository(Campaign::class)
        ->createQueryBuilder('c')
        ->where('c.id IN (:ids)')
        ->setParameter('ids', $campaignIds)
        ->getQuery()
        ->getResult();
}
```

**Integration in `getActiveBiddingRounds`:**

```php
// After existing cross-promotion lookup (line 141-169)
// Additional: find campaigns where student is included via module_config JSON
$jsonCrossPromotionCampaigns = $this->campaignPhaseConfigRepository
    ->findOpenCampaignsByStudentIncludeInConfig((string) $student->getId());

foreach ($jsonCrossPromotionCampaigns as $campaign) {
    if (in_array($campaign->getId(), $existingCampaignIds, true)) {
        continue;
    }
    // ... same eligibility check pattern as existing cross-promotion ...
}
```

**Why LIKE-based search is acceptable:**
- This is a read-only query for student dashboard loading
- The result set is already filtered by campaign status, dates, and deletion flags
- The LIKE pattern is further validated by `isStudentEligibleForCampaign` after discovery
- False positives are harmless (just trigger an eligibility check that returns false)

## Risks / Trade-offs

**[Risk] LIKE-based JSON search may have false positives**
→ **Mitigation:** The `isStudentEligibleForCampaign` check (already in place) fully validates eligibility after discovery. False positives from the LIKE query are filtered out by the eligibility check.

**[Risk] Performance of JSON LIKE search on large datasets**
→ **Mitigation:** The query is already filtered by `c.status = 'open'`, `c.isDeleted = false`, and `c.endDate >= NOW()`, limiting the result set. For the current scale this is acceptable.

**[Risk] Existing filter merge fix becomes unreachable if discovery fails**
→ **Mitigation:** This design specifically fixes the discovery step, making the existing merge fix effective.

**[Trade-off] Dual-storage system remains in place**
→ A unification of the storage system (migrating JSON filters to DB table records) would be a cleaner long-term solution, but is out of scope for this bug fix.

## Migration Plan

1. **Code change** — Add `findOpenCampaignsByStudentIncludeInConfig` repository method
2. **Code change** — Update `ActiveCampaignService::getActiveBiddingRounds` to call the new lookup
3. **No database migration required** — querying existing JSON data
4. **Deployment:** Standard PHP deployment
5. **Rollback:** Simple code revert if issues arise
6. **Testing:**
   - Create campaign for promotion A with Pre-bidding module
   - Add student from promotion B to Pre-bidding ONLY (via module config, not campaign-level)
   - Impersonate as student B → verify campaign appears on dashboard
   - Advance to Bidding Round phase → verify campaign does NOT appear for student B
