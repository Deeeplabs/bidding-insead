## Context

In the current live system, updating the main configuration for a campaign (e.g., editing `min_total_points` or other top-level properties on the Campaign edit screen) inadvertently resets active phases (such as an ongoing Bidding Round) back to a "Not Started" state. This effectively masks submitted bids and locks out students. Furthermore, attempting to progress past a corrupted phase causes the Simulation phase to be entirely skipped, instantly jumping the campaign to a closed Final Enrollment state. The goal is to decouple top-level campaign configuration updates from phase status coercions and fix the broken phase progression logic.

## Goals / Non-Goals

**Goals:**
- Correctly scope the update operations inside the campaign update service so that general updates do not defensively or inadvertently clear child phase statuses to `Not Started` or `0` states.
- Ensure the admin action to edit global campaign rules does not forcefully interfere with actively running modular phases.
- Fix the bug causing the UI or API to skip simulation phases and prematurely jump to "Final Enrollment" due to inconsistent prior state markers.

**Non-Goals:**
- NOT an overhaul of the entire Session or Campaign state machine.
- NOT making any schema (database entity) modifications.
- NOT removing or structurally altering REST API endpoint JSON contracts.

## Decisions

- **Isolate Phase Configuration Saves:** When an update is triggered for a specific campaign module via the API, the backend service must correctly identify which module is being updated and apply mutations *only* to the associated properties. We must decouple or branch the phase-reset logic that currently runs indiscriminately whenever any module is saved.
- **Verification of Phase Boundaries:** Modify the logic that computes/updates the parent `Campaign` phase (or session status) to prevent closing active rounds unless explicitly closed by the user or scheduled chronologically.
- **Simulation Jump Mitigation:** The jump to Final Enrollment logic should be audited to verify that it correctly waits for an explicit Simulation status of "Completed" and not a false positive from the Bidding phase prematurely closing.

## Risks / Trade-offs

- **[Risk]** Unintentional side-effects on the simulation engine if the campaign state machine logic relies heavily on the buggy behavior to advance progress.
  - **Mitigation:** Rely on thorough manual testing to confirm active status correctly persists across updates for each phase transition. Fix the specific status coercion without rewriting the complete state transition logic.
