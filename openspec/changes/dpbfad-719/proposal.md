## Why

When a campaign is duplicated with "Duplicate all configurations of Bidding Rounds" enabled, the `CampaignPhaseConfig` records are cloned with their `isActive` flag and date ranges copied from the source campaign. The `CampaignService` then computes module `is_active` status based on these date ranges — so if the current date falls within a copied date range, the module appears as "OPEN" in the duplicated (Draft) campaign. This is incorrect: a Draft campaign should have all modules in CLOSE state, since the PM has not explicitly activated anything.

## What Changes

- During campaign duplication, set `CampaignPhaseConfig.isActive` to `false` instead of copying from the source campaign, ensuring all phase configs in the new campaign start inactive.
- In `CampaignService` module status computation, enforce that Draft campaigns always report `is_active = 'CLOSE'` for all modules, regardless of date ranges. This prevents date-based activation logic from applying to campaigns that haven't been launched.
- In `CampaignToModuleDetailDtoMapper` (student-facing), apply the same Draft enforcement so students never see active modules in Draft campaigns.

## Capabilities

### New Capabilities
- `draft-campaign-module-status`: Enforce that all modules in Draft campaigns have `is_active = CLOSE`, bypassing date-range-based status computation. Applies to both duplication and general Draft campaign behavior.

### Modified Capabilities

## Impact

- **bidding-api**: `CampaignDuplicationService` (duplication logic), `CampaignService` (admin module status computation), `CampaignToModuleDetailDtoMapper` (student module status computation)
- **Entities affected**: `CampaignPhaseConfig` (isActive handling during duplication)
- **No migration required**: No schema changes — this is a behavioral fix in PHP service logic
- **No frontend changes**: The admin and student UIs already render based on the `is_active` string returned by the API; fixing the API response is sufficient
- **Backward compatibility**: Existing non-Draft campaigns are unaffected. Only Draft campaigns (including newly duplicated ones) will have enforced CLOSE status.
