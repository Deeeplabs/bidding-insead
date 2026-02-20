## Why

A student from a different promotion who is added to a campaign for "Pre-bidding ONLY" cannot see the campaign on their student dashboard or bidding page. The previous fix (merging `tableFilters` and `configFilters` in `isStudentEligibleForCampaign`) was necessary but insufficient — the campaign is never even discovered for cross-promotion students because the discovery mechanism (`findOpenCampaignsByStudentInclude`) searches only the `CampaignStudentFilter` database table, while per-module student includes are stored exclusively in `module_config.config.student_filters` JSON blob.

**The actual flow has TWO problems:**

1. **Discovery Gap (PRIMARY):** `ActiveCampaignService::getActiveBiddingRounds` discovers cross-promotion campaigns via `CampaignStudentFilterRepository::findOpenCampaignsByStudentInclude`, which queries the `campaign_student_filters` table. But when admins configure per-module student includes (e.g., "Pre-bidding ONLY"), those filters are saved to `CampaignPhaseConfig.module_config` JSON only — never to the `campaign_student_filters` table. So the campaign is never discovered at all.

2. **Eligibility Check (ALREADY FIXED):** `isStudentEligibleForCampaign` merges `tableFilters` and `configFilters` — this was previously fixed but is moot if the campaign is never discovered in step 1.

## What Changes

- Modify `ActiveCampaignService::getActiveBiddingRounds` to add a secondary cross-promotion lookup that also searches `module_config` JSON in `CampaignPhaseConfig` for `student_include` entries containing the student's ID.
- Add a new repository method `CampaignPhaseConfigRepository::findOpenCampaignsByStudentIncludeInConfig` (or extend the existing lookup) to query JSON-stored student includes.

## Capabilities

### New Capabilities
- None — this is a bug fix

### Modified Capabilities
- `student-campaign-eligibility`: Fix cross-promotion campaign discovery to also search module_config JSON for student_include entries

## Impact

**Affected Code:**
- `bidding-api/src/Domain/Campaign/ActiveCampaign/ActiveCampaignService.php` — `getActiveBiddingRounds` method: add JSON-based cross-promotion lookup
- `bidding-api/src/Repository/CampaignStudentFilterRepository.php` — `findOpenCampaignsByStudentInclude` or a new companion method to search JSON config
- `bidding-api/src/Domain/Campaign/ActiveCampaign/CampaignStudentEligibilityService.php` — already has the merge fix, no further changes needed

**Testing:**
- Verify that students added via per-module `student_include` (JSON config) can see the campaign on their dashboard
- Verify cross-promotion scenario: student from promotion B added to campaign of promotion A for Pre-bidding ONLY
- Verify students NOT in any includeList are still properly handled

**No Breaking Changes:**
- This is a bug fix that corrects incomplete behavior
- No existing API contracts are changed
- No database migrations required
