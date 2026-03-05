## Context

In the admin dashboard, PMs can duplicate bidding rounds. The original bidding round has associated campaigns/modules which maintain status states, like "filter" for early operations. Currently, duplicating these bidding rounds indiscriminately copies the previous status attached to the modules even when the PM didn't duplicate the associated course list. This inherits stale module statuses (like "filter" or incorrect start/end dates) into the newly duplicated bidding round module. Furthermore, module-specific course filters are retained even when the PM chooses not to duplicate the course list. Since the PM can manually control (start/end) the module, the status shown to users shouldn't incorrectly display the old status. The module should accurately reflect its own scheduled timelines or manual overrides, e.g. "Actual Start–End Date (Expected Start–End Date)" when altered.

## Goals / Non-Goals

**Goals:**
- Prevent copy-over of statuses when campaigns/modules are duplicated through the admin dashboard module clone process.
- Ensure that if the course list is not duplicated, module-level course filters and selections are cleared.
- Display manual actual start/end dates alongside predefined expected schedules on the PM module dashboard/list, formatting as: `Actual Start–End Date (Expected Start–End Date)`.

**Non-Goals:**
- Completely rewriting campaign duplication flows.
- Touching course list duplication logics apart from verifying what statuses are attached and clearing module-level overrides when required.

## Decisions

- **Decision 1: Exclude status fields during duplication**
  When a module or bidding phase is duplicated using `CampaignDuplicateService` (or equivalent cloning flow), ensure that stateful tracking properties like `status`, `actualStartDate`, `actualEndDate` are reset or excluded. This ensures the new module inherits clean expected states based on the duplicated configurations.

- **Decision 2: Clear course-related configurations when duplicating without courses**
  If `duplicateProgrammeCourses` is false, we must actively strip `course_filters`, `course_selection`, and `selected_course_ids` from the module configuration JSON for both `CampaignModule` and `CampaignPhaseConfig`.

- **Decision 3: Enhance Response Transformers**
  In the API, when exposing a module/phase detail where an actual manual override has been engaged, the dates output will continue to return both variables. On the admin dashboard, enhance mapping/display logics to print `[Actual Start-End Date] ([Expected Start-End Date])` dynamically if the strings differ or if an actual date is populated overriding expected dates.

## Risks / Trade-offs

- **Risk: Breaking PM dashboard expectations of identical copies** -> **Mitigation**: Status states shouldn't rationally copy because the new round hasn't started yet. The dates will reset. This is intended functionality.
- **Risk: Discrepancy with Student Portal dates** -> **Mitigation**: Wait, if the status actually resets, student portals will just see the expected dates until manual trigger occurs. No negative outcome.
