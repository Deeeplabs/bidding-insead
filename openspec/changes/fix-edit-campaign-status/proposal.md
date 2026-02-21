# Proposal: Fix Campaign Status Reversion and Module Closing Bug

## Problem Statement

When a user edits a campaign through the admin portal while a bidding round is actively running, the campaign module (Bidding Round) reverts to its original draft-like state. This unintentionally overrides the running phase's `start_date` and `end_date` back to their initial configured dates, causing the phase to interpret itself as `CLOSE` and effectively terminating the current round.

During our investigation, we identified the root cause in the backend API: when a phase is opened (via `CampaignPhaseService::activatePhase()`), the `CampaignPhaseConfig` database column `start_date` is updated with the current activation timestamp. However, the JSON blob `module_config` remains stale holding the original unactivated date config. 

When a user updates a separate module (such as the Simulation phase), the `PUT /v2/api/campaigns/{id}/save-with-modules` endpoint is hit with the full campaign context payload. The frontend fetches and passes along the Bidding Round's stale `module_config` straight into the `updateWithModules` method, which blindly synchronizes the `CampaignPhaseConfig` `start_date` back into the past or future date, forcing the frontend into a `CLOSE` mapping.

## Proposed Solution

Update the JSON value within `module_config` inside `CampaignPhaseService::activatePhase()` whenever a phase's start status (`open`) or end status (`close`) runs. Consequently, when the frontend fetches the detailed state of the campaign, it receives the correctly synchronized JSON data. Any future PUT requests sent to `save-with-modules` will now carry over the proper live dates, stopping the destructive overwriting effect onto the active phase configuration.
