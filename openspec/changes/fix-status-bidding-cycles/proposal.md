## Why

When the main campaign configuration is updated (e.g., modifying "Minimum total points" via the campaign Edit screen), any currently active phases (like a Bidding Round) are incorrectly reset to "Not Started". This causes associated statistics (e.g., submitted student bids) to display as 0, effectively hiding or invalidating the data. Furthermore, progressing past the disrupted bidding phase causes the next phase (Simulation) to be skipped entirely, immediately transitioning the campaign directly into a "Closed" Final Enrollment state. This state-management violation is critical as it corrupts live campaign data and prevents administrators from properly running the simulation and add/drop phases.

## What Changes

- Modify the main campaign update logic so that editing campaign-wide settings (e.g., minimum points, max credits) does not coerce, clear, or reset the status of existing campaign modules/phases to "Not Started".
- Ensure that an active module (e.g., Bidding Round) retains its exact state and persisted data points (bid counts, enrollments) across unrelated campaign updates.
- Fix the phase transition bug that causes the system to skip the Simulation phase and prematurely jump to the "Final Enrollment [CLOSED]" state.
- Ensure only an explicit closure of a phase transitions it from "Active" to its completed equivalent state.

## Capabilities

### New Capabilities
- `fix-status-bidding-cycles`: Ensures campaign configuration updates correctly isolate state and prevent unintentional side effects on unrelated active campaign phases.

### Modified Capabilities

- 

## Impact

- **bidding-api**: Will likely involve changes in campaign configuration persistency handlers, specifically involving `CampaignPhaseService`, `CampaignPhaseConfig` operations, and potentially session lifecycle modules.
- **Entities**: Does not require new entities, but updates to state-check logic prior to storing campaign phase configurations.
- **Backward Compatibility**: High priority. This fix ensures live campaigns retain data integrity when admins update parameters without destroying current active states.
