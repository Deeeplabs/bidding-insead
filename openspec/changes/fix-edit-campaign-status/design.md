# Design: Fix Campaign Status Reversion and Module Closing Bug

## Objective

Fix an issue where campaign modules revert their status and forcibly close bidding rounds when bulk-updating the campaign. Ensure that the active and closed states from the backend accurately flow down symmetrically between the entity properties and the embedded configuration blob (`module_config`).

## Root Cause Analysis

1. **State Divergence:** The `CampaignPhaseService::activatePhase` function correctly modifies the `CampaignPhaseConfig`'s scalar database columns (`start_date`, `end_date`), but leaves its internal JSON `module_config` attribute to still point to its original setup value.
2. **Frontend Misinterpretation over Updates:** The frontend UI relies heavily on the full campaign config payload. Because `module_config` contains the stale properties, editing *any other module* bundles the old attributes.
3. **Backend Overwriting Constraint:** When `PUT /v2/api/campaigns/{id}/save-with-modules` uses `updateWithModules`, it reads the stale dates given by `module_config`. It overwrites the column using the exact old dates, reverting the phase effectively.

## Architecture & Modifications 

### API Updates

*   **`CampaignPhaseService.php` (`activatePhase`)**
    *   **Open Operations:** After calling `setStartDate($now)`, fetch the JSON object via `getModuleConfig()`. Add or update the key `start_date` to map the exact `$now` using `\DateTime::ATOM`. Call `setModuleConfig($moduleConfig)`.
    *   **Close Operations:** After calling `setEndDate($now)`, fetch the JSON object via `getModuleConfig()`. Add or update the key `end_date` to map the exact `$now` using `\DateTime::ATOM`. Call `setModuleConfig($moduleConfig)`.

### Downstream Effects

*   **Idempotent Safety:** Because `updateWithModules` replaces database dates safely to what it reads from the config, doing this will just securely replace timestamps with what they already are—meaning the entity doesn't shift its state properties away from Open.
*   **Performance:** A fast mapping mutation since the JSON object is natively an array in PHP and Doctrine gracefully handles `json` database mapping. No performance bottlenecks.
*   **Compatibility:** Admin UI logic seamlessly accommodates this because the format matches the standard `DateHelper::toIso` standard structure. No frontend modifications are needed.
