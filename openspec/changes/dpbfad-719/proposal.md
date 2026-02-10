## Why

Two related issues with campaign status and module activation:

1. **Duplication bug**: When a campaign is duplicated with "Duplicate all configurations of Bidding Rounds" enabled, the `CampaignPhaseConfig` records are cloned with their `isActive` flag and date ranges from the source campaign. If copied date ranges span the current date, modules appear as "OPEN" in the duplicated (Draft) campaign. Draft campaigns should have all modules in CLOSE state since the PM has not explicitly activated anything.

2. **Missing auto-transition**: Draft status means the PM hasn't done the configuration in the core campaign. When the PM starts any module (via `PUT /dashboard/campaign/status`, i.e. `CampaignPhaseService::activatePhase()`), the campaign status should automatically transition from `draft` to `open`. Currently `activatePhase` only sets phase `startDate` without touching the campaign status, so the campaign remains in `draft` and the draft guard immediately forces all modules back to CLOSE — making it impossible to open modules in a draft campaign.

## What Changes

- During campaign duplication, set `CampaignPhaseConfig.isActive` to `false` instead of copying from the source campaign, ensuring all phase configs in the new campaign start inactive.
- In `CampaignService` module status computation, enforce that Draft campaigns always report `is_active = 'CLOSE'` for all modules, regardless of date ranges.
- In `CampaignToModuleDetailDtoMapper` and `CampaignToActiveBiddingRoundDtoMapper` (student-facing), apply the same Draft enforcement.
- In `CampaignPhaseService::activatePhase()`, when a PM opens a module phase and the campaign is in `draft` status, automatically transition the campaign status to `open`. This ensures the draft guard is lifted as soon as the PM starts configuring modules.

## Capabilities

### New Capabilities
- `draft-campaign-module-status`: Enforce that all modules in Draft campaigns have `is_active = CLOSE`, bypassing date-range-based status computation. When a PM opens any module, automatically transition the campaign from `draft` to `open`. Applies to both duplication and general Draft campaign behavior.

### Modified Capabilities

## Impact

- **bidding-api**: `CampaignDuplicationService` (duplication logic), `CampaignService` (admin module status computation), `CampaignToModuleDetailDtoMapper` (student module status), `CampaignToActiveBiddingRoundDtoMapper` (student bidding round status), `CampaignPhaseService` (auto-transition on module activation)
- **Entities affected**: `CampaignPhaseConfig` (isActive handling during duplication), `Campaign` (status auto-transition)
- **No migration required**: No schema changes — this is a behavioral fix in PHP service logic
- **No frontend changes**: The admin and student UIs already render based on the `is_active` string returned by the API; fixing the API response is sufficient
- **Backward compatibility**: Existing non-Draft campaigns are unaffected. Only Draft campaigns (including newly duplicated ones) will have enforced CLOSE status. The auto-transition is a no-op for campaigns already in `open` or `close` status.
