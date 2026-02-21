# Fix Campaign Status Reversion and Module Closing Bug

## Problem

When a user edits a campaign through the admin portal while a bidding round is actively running, the campaign module (Bidding Round) was reverting to its original draft-like state. This unintentionally overrode the running phase's `start_date` and `end_date` back to their initial configured dates, causing the phase to interpret itself as `CLOSE` and effectively terminating the current round.

### Root Cause Analysis

1. **State Divergence:** The `CampaignPhaseService::activatePhase` function correctly modified the `CampaignPhaseConfig`'s scalar database columns (`start_date`, `end_date`), but left its internal JSON `module_config` attribute to still point to its original setup value.
2. **Frontend Misinterpretation over Updates:** The frontend UI relies heavily on the full campaign config payload. Because `module_config` contained the stale properties, editing *any other module* bundled the old attributes and pushed them back to the backend.
3. **Backend Overwriting Constraint:** When `PUT /v2/api/campaigns/{id}/save-with-modules` used `updateWithModules`, it read the stale dates given by `module_config`. It overwrote the column using the exact old dates, completely reverting the phase payload safely.

## Solution

Update the JSON value within `module_config` inside `CampaignPhaseService::activatePhase()` whenever a phase's start status (`open`) or end status (`close`) runs. Consequently, when the frontend fetches the detailed state of the campaign, it receives the correctly synchronized JSON data. Any future PUT requests sent to `save-with-modules` will now carry over the proper live dates, stopping the destructive overwriting effect onto the active phase configuration.

### Changes Made

**Modified File:** `src/Domain/Campaign/Campaign/CampaignPhaseService.php`

- In the `activatePhase` method, when handling the `open` status, after setting the scalar `start_date`, we now also fetch the JSON `module_config` array and explicitly update the string value for `start_date` utilizing the precise `\DateTime::ATOM` format representing `$now`.
- Similarly, when handling the `close` status, after setting the scalar `end_date`, we fetch and explicitly update the `end_date` value inside `module_config` using `\DateTime::ATOM` representing `$now`.
- The parsed array is then re-set onto the `$phaseConfig->setModuleConfig($moduleConfig)` function, accurately recording it back to the database state upon execution flush.

## Files Changed

| File | Change |
|------|--------|
| `src/Domain/Campaign/Campaign/CampaignPhaseService.php` | Synchronize \DateTime::ATOM payload into JSON module_config properties upon activating/closing phases. |

## Impact

*   **Idempotent Safety:** Because `updateWithModules` replaces database dates safely to what it reads from the config, doing this will just securely replace timestamps with what they already are—meaning the entity doesn't shift its state properties away from Open.
*   **Performance:** A fast mapping mutation since the JSON object is natively an array in PHP and Doctrine gracefully handles `json` database mapping. No performance bottlenecks.
*   **Compatibility:** Admin UI logic seamlessly accommodates this because the format matches the standard `DateHelper::toIso` standard structure. No frontend modifications are needed.

## Testing

Verified through comprehensive API testing:
- [x] Tested initiating a Bidding Round and verified the `module_config` properly displays identical `start_date` matching the entity properties.
- [x] Verified editing a distinct module component (like Simulation) doesn't interrupt or forcefully reset the active Bidding Round boundaries utilizing identical payload boundaries.
- [x] Asserted no downstream database constraint exceptions occur during payload submission sequences.
