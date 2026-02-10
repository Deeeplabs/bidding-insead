## DPBFAD-719: Fix Duplicate Campaign copying module status incorrectly

### Problem
**Jira**: https://insead.atlassian.net/browse/DPBFAD-719

When duplicating a campaign with "Duplicate all configurations of Bidding Rounds" enabled, module statuses (OPEN/COMPLETED) from the source campaign are carried over to the new Draft campaign. This happens because:

1. `CampaignDuplicationService` copies `CampaignPhaseConfig.isActive` from the source
2. Status computation in `CampaignService`, `CampaignToModuleDetailDtoMapper`, and `CampaignToActiveBiddingRoundDtoMapper` evaluates copied date ranges against the current date, causing modules to appear OPEN in a Draft campaign

### Solution

Two-layered fix:

**Layer 1 — Duplication**: Force `isActive = false` on all cloned `CampaignPhaseConfig` records instead of copying from source.

**Layer 2 — Draft guard**: Add early-return guards in all 4 status computation locations so Draft campaigns always return `CLOSE`/`false` for module status, regardless of date ranges.

### Changes

| File | Change |
|------|--------|
| `src/Domain/Campaign/Campaign/CampaignDuplicationService.php` | `setIsActive(false)` instead of `setIsActive($sourcePhaseConfig->isActive())` |
| `src/Domain/Campaign/Campaign/CampaignService.php` | Draft guard in `mapCampaignToDto` closure and `mapCampaignToDetailDto` execution loop |
| `src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToModuleDetailDtoMapper.php` | Draft guard in `buildModulesForBiddingRound` — `is_active = false` for draft |
| `src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapper.php` | Draft guard in `buildModuleItem` — `is_active = false` for draft |
| `tests/Unit/Domain/Campaign/Campaign/CampaignDuplicationServiceTest.php` | New test: verifies duplicated phase configs have `isActive = false` |
| `tests/Unit/Domain/Campaign/Campaign/CampaignServiceTest.php` | New test: verifies draft campaign returns `CLOSE` for all modules; updated setUp to match current constructor |

### Testing

- **Unit**: 2 new PHPUnit tests covering both fix layers
- **Manual**: Duplicate a campaign with active bidding rounds → verify all modules show CLOSE in the new Draft campaign

### Impact

- No migration needed — behavioral fix only
- No frontend changes — admin and student UIs already render based on the `is_active` string from the API
- Non-Draft campaigns are unaffected
