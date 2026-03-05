## Why

When duplicating bidding rounds, the process incorrectly copies the module status and configurations. Specifically:
1. It copies stale module statuses from the former campaign. Because the PM can manually start the modules, the status should accurately reflect the new module's actual dates and status.
2. When duplicating a campaign without duplicating the course list (`duplicateProgrammeCourses` is false), the new modules inappropriately inherit the "course filters" from the former campaign's module configurations, causing them to show old filters.

## What Changes

- Update the campaign duplication logic so that it does not inappropriately carry over old module statuses when duplicating bidding rounds.
- Enhance the duplication logic to explicitly clear `course_filters`, `course_selection`, and `selected_course_ids` inside the module and phase configurations if the PM chooses not to duplicate the course list.
- When the PM chooses to start or end the module earlier, update the display to show:
  Actual Start–End Date (Expected Start–End Date).
- Ensure that the dashboard and module list accurately reflect the current, real-time status and manually triggered status updates, rather than the stale status inherited from the copied campaign.

## Capabilities

### New Capabilities

- `fix-duplicate-module-status`: Ensures module statuses are not copied over incorrectly during duplication, and correctly displays manual start/end dates if different from expected dates.

### Modified Capabilities


## Impact

- **bidding-api**: Needs an update in the Campaign duplication logic (`CampaignDuplicateService` or equivalent) to clear or reset the status of copied modules. Modifying how dates and states are served to the frontend during campaign retrieval.
- **bidding-admin**: Update the campaign and module components to display "Actual Date (Expected Date)" if the module was manually started or ended early.
- **bidding-web**: Potential impact if students view course status dates; requires alignment.
- **Entities**: Adjusting `CampaignModule` or `Session` data states during duplication.
