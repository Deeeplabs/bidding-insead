## Context

Campaign duplication creates a new campaign in Draft status. The duplication service (`CampaignDuplicationService`) clones `CampaignPhaseConfig` records including their `isActive` flag and date ranges from the source campaign. The admin API (`CampaignService`) and student-facing mappers (`CampaignToModuleDetailDtoMapper`, `CampaignToActiveBiddingRoundDtoMapper`) compute module `is_active` status strings (`OPEN`/`CLOSE`/`COMPLETED`) based on whether the current date falls within a phase config's date range. When copied dates span the current date, modules incorrectly appear as `OPEN` in a Draft campaign.

Current module status computation in `CampaignService` (lines 424-430):
```
if start_date < today < end_date → 'OPEN'
if start_date > today → 'CLOSE'
else → 'COMPLETED'
```

This date-based logic runs regardless of campaign status, so Draft campaigns with copied date ranges can show active modules.

Additionally, when the PM activates a module phase via `PUT /dashboard/campaign/status` (handled by `CampaignPhaseService::activatePhase()`), the method only sets the phase's `startDate` to now — it does **not** transition the campaign status from `draft` to `open`. Because the Draft guard forces all modules to `CLOSE`, starting a module in a draft campaign has no visible effect: the phase gets a start date, but the guard still overrides the status to CLOSE. The campaign must be explicitly opened via `CampaignExecutionService::startCampaign()` first, which is a separate action. The expected behavior (per product) is: Draft = PM hasn't done the configuration in the core campaign; once the PM starts any module, the campaign should automatically transition to `open`.

## Goals / Non-Goals

**Goals:**
- Ensure all modules in duplicated campaigns have `is_active = 'CLOSE'`
- Ensure all modules in any Draft campaign have `is_active = 'CLOSE'`, regardless of phase config date ranges
- Automatically transition campaign status from `draft` to `open` when the PM opens any module via `activatePhase`
- Fix at the API layer so both admin and student frontends automatically reflect the correct state

**Non-Goals:**
- Changing the duplication dialog UI or toggle options
- Modifying how non-Draft campaigns compute module status
- Changing the `CampaignModule.isActive` entity default (boolean) — the string status is what matters to the frontends
- Resetting or clearing copied date ranges (they should be preserved for when the PM activates the campaign)
- Changing `CampaignExecutionService::startCampaign()` — it remains available as an explicit campaign start action

## Decisions

### Decision 1: Fix at duplication point AND add Draft-status guard

**Approach**: Two-layered fix:

1. **Duplication layer** (`CampaignDuplicationService`): When cloning `CampaignPhaseConfig` records, set `isActive` to `false` instead of copying from source. This ensures the duplicated campaign's phase configs start inactive.

2. **Status computation layer** (`CampaignService` + `CampaignToModuleDetailDtoMapper` + `CampaignToActiveBiddingRoundDtoMapper`): Add an early return that checks if the campaign's status is Draft. If Draft, return `'CLOSE'` for all modules without evaluating date ranges.

**Why both layers?**
- The duplication fix addresses the immediate bug
- The Draft-status guard is a defensive measure: if phase configs are ever activated accidentally (e.g., through a direct DB edit or future code path), Draft campaigns will still show all modules as CLOSE
- The guard also protects campaigns that are manually set back to Draft status

**Alternatives considered:**
- *Only fix duplication*: Simpler but fragile — any other code path that sets `isActive = true` on a Draft campaign's phase config would re-introduce the bug
- *Only add Draft guard*: Would fix the display but leave `isActive = true` on phase configs, potentially causing issues if other code reads this flag directly
- *Clear dates during duplication*: Rejected — dates should be preserved as a template for when the PM is ready to activate

### Decision 2: Auto-transition campaign from draft to open on module activation

**Approach**: In `CampaignPhaseService::activatePhase()`, after successfully setting the phase `startDate` (when `$status === 'open'`), check if the campaign is in `draft` status. If so, automatically set `$campaign->setStatus('open')` and persist. This happens before `$this->entityManager->flush()` so both changes are committed together.

**Why?**
- The `PUT /dashboard/campaign/status` endpoint (activatePhase) is the action the PM uses to start modules. This is the natural trigger for transitioning the campaign from draft to open.
- Without this, the Draft guard makes it impossible to open modules from a draft campaign: `activatePhase` sets the start date but the guard immediately overrides `is_active` to CLOSE because the campaign is still draft.
- This aligns with the product expectation: "draft is when the PM hasn't done the configuration in the core campaign" — once they start configuring (opening modules), the campaign should be open.

**Alternatives considered:**
- *Require explicit `startCampaign` call first*: Adds an extra step for the PM and doesn't match the expected workflow.
- *Remove the Draft guard entirely*: Would fix the immediate issue but lose the protective behavior for genuinely draft campaigns (e.g., right after duplication).

### Decision 3: Campaign status check mechanism

The campaign status is stored directly on the `Campaign` entity as a string field (`$status = 'draft'`). Check `$campaign->getStatus() === 'draft'`. The valid statuses are: `draft`, `open`, `close` (see `CampaignValidationService::STATUS_TRANSITIONS`).

### Decision 4: Apply fix to both admin and student mappers

Both `CampaignService` (admin-facing module list) and `CampaignToModuleDetailDtoMapper` + `CampaignToActiveBiddingRoundDtoMapper` (student-facing) compute `is_active` from date ranges. All three need the Draft guard.

## Risks / Trade-offs

- **[Risk] Auto-transition side effects** → When `activatePhase` auto-transitions a campaign to `open`, any logic gated on campaign status `open` will activate (e.g., `validatePhaseSequencing` enforces sequencing only for `open`/`closed` campaigns). This is correct behavior — once modules are being started, sequencing should be enforced.
- **[Risk] Other code paths reading `CampaignPhaseConfig.isActive` directly** → The duplication fix (setting to false) mitigates this. A codebase search for `.isActive()` on phase configs should confirm no other consumers are affected.
- **[Risk] Campaigns set back to Draft after being Open** → The Draft guard will force all modules to CLOSE. The PM can re-open modules which will auto-transition the campaign back to open. This is the desired behavior.
- **[Trade-off] Three-layer fix vs single fix** → More code, but significantly more robust. The guard ensures correctness for display, the duplication fix ensures data correctness, and the auto-transition ensures correct workflow.
