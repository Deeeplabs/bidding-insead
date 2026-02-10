## DPBFAD-719: Fix Duplicate Campaign copying module status incorrectly + Auto-transition draft→open

### Problem
**Jira**: https://insead.atlassian.net/browse/DPBFAD-719

Two related issues with campaign status and module activation:

1. **Duplication bug**: `CampaignDuplicationService` copies `CampaignPhaseConfig.isActive` from the source. Status computation in `CampaignService`, `CampaignToModuleDetailDtoMapper`, and `CampaignToActiveBiddingRoundDtoMapper` evaluates copied date ranges against the current date, causing modules to appear OPEN in a Draft campaign.

2. **Missing auto-transition**: Draft status means the PM hasn't configured the core campaign yet. When the PM starts any module via `PUT /dashboard/campaign/status` (`CampaignPhaseService::activatePhase()`), the campaign should automatically transition from `draft` to `open`. Without this, the draft guard forces all modules to CLOSE even after the PM explicitly opens one — making it impossible to activate modules from a draft campaign.

### Solution

Three-layered fix:

**Layer 1 — Duplication**: Force `isActive = false` on all cloned `CampaignPhaseConfig` records instead of copying from source.

**Layer 2 — Draft guard**: Add early-return guards in all 4 status computation locations so Draft campaigns always return `CLOSE`/`false` for module status, regardless of date ranges.

**Layer 3 — Auto-transition**: In `CampaignPhaseService::activatePhase()`, when the PM opens a phase and the campaign is in `draft` status, automatically transition the campaign to `open`. This lifts the draft guard so the opened module is immediately visible as OPEN.

### Changes

| File | Change |
|------|--------|
| `src/Domain/Campaign/Campaign/CampaignDuplicationService.php` | `setIsActive(false)` instead of `setIsActive($sourcePhaseConfig->isActive())` |
| `src/Domain/Campaign/Campaign/CampaignService.php` | Draft guard in `mapCampaignToDto` closure and `mapCampaignToDetailDto` execution loop |
| `src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToModuleDetailDtoMapper.php` | Draft guard in `buildModulesForBiddingRound` — `is_active = false` for draft |
| `src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapper.php` | Draft guard in `buildModuleItem` — `is_active = false` for draft |
| `src/Domain/Campaign/Campaign/CampaignPhaseService.php` | Auto-transition: after `setStartDate($now)` in `activatePhase()`, check if campaign is `draft` and set to `open` |
| `tests/Unit/Domain/Campaign/Campaign/CampaignDuplicationServiceTest.php` | New test: verifies duplicated phase configs have `isActive = false` |
| `tests/Unit/Domain/Campaign/Campaign/CampaignServiceTest.php` | New test: verifies draft campaign returns `CLOSE` for all modules; updated setUp to match current constructor |
| `tests/Unit/Domain/Campaign/Campaign/CampaignPhaseServiceTest.php` | 3 new tests: draft→open transition on phase open, no change when already open, no change on phase close; updated setUp to match current constructor |

### Testing

- **Unit**: 5 new PHPUnit tests covering all three fix layers
- **Manual**:
  - Duplicate a campaign with active bidding rounds → verify all modules show CLOSE in the new Draft campaign
  - Open a module on the duplicated (draft) campaign → verify campaign transitions to `open` and the module shows as OPEN
  - Open a module on an already-open campaign → verify no status change
  - Close a module → verify campaign status is unchanged

### Impact

- No migration needed — behavioral fix only
- No frontend changes — admin and student UIs already render based on the `is_active` string from the API
- Non-Draft campaigns are unaffected
- Auto-transition only fires for `draft` → `open`; campaigns in `open` or `close` status are not touched
