# Tasks: Fix Campaign Status Reversion and Module Closing Bug

## 1. Backend Fixes (API)
- [x] **Update `activatePhase()` inside `CampaignPhaseService.php` to push JSON state logic.**  
  **Details:** Modify `activatePhase` to map and update the string property `start_date` and `end_date` inside `$phaseConfig->getModuleConfig()` when opening and closing respectively. Use the `\DateTime::ATOM` format for precise synchronization back to the caller interface. *(Already completed during investigation)*

## 2. Validation
- [x] **Verify End-to-End Status Integrity:**
  **Details:** Editing a different campaign module while a campaign Bidding Round or Pre-Bidding Round is running must NOT automatically suspend, abort, or reset the currently run state of the execution round. The UI should consistently maintain `OPEN` status.
- [x] **Verify Exception Cleanliness:**
  **Details:** `PUT /v2/api/campaigns/{id}/save-with-modules` proceeds flawlessly without throwing database constraints exceptions or overriding constraints.
