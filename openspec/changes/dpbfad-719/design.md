## Context

Campaign duplication creates a new campaign in Draft status. The duplication service (`CampaignDuplicationService`) clones `CampaignPhaseConfig` records including their `isActive` flag and date ranges from the source campaign. The admin API (`CampaignService`) and student-facing mapper (`CampaignToModuleDetailDtoMapper`) compute module `is_active` status strings (`OPEN`/`CLOSE`/`COMPLETED`) based on whether the current date falls within a phase config's date range. When copied dates span the current date, modules incorrectly appear as `OPEN` in a Draft campaign.

Current module status computation in `CampaignService` (lines 424-430):
```
if start_date < today < end_date → 'OPEN'
if start_date > today → 'CLOSE'
else → 'COMPLETED'
```

This date-based logic runs regardless of campaign status, so Draft campaigns with copied date ranges can show active modules.

## Goals / Non-Goals

**Goals:**
- Ensure all modules in duplicated campaigns have `is_active = 'CLOSE'`
- Ensure all modules in any Draft campaign have `is_active = 'CLOSE'`, regardless of phase config date ranges
- Fix at the API layer so both admin and student frontends automatically reflect the correct state

**Non-Goals:**
- Changing the duplication dialog UI or toggle options
- Modifying how non-Draft campaigns compute module status
- Changing the `CampaignModule.isActive` entity default (boolean) — the string status is what matters to the frontends
- Resetting or clearing copied date ranges (they should be preserved for when the PM activates the campaign)

## Decisions

### Decision 1: Fix at duplication point AND add Draft-status guard

**Approach**: Two-layered fix:

1. **Duplication layer** (`CampaignDuplicationService`): When cloning `CampaignPhaseConfig` records, set `isActive` to `false` instead of copying from source. This ensures the duplicated campaign's phase configs start inactive.

2. **Status computation layer** (`CampaignService` + `CampaignToModuleDetailDtoMapper`): Add an early return that checks if the campaign's status is Draft. If Draft, return `'CLOSE'` for all modules without evaluating date ranges.

**Why both layers?**
- The duplication fix addresses the immediate bug
- The Draft-status guard is a defensive measure: if phase configs are ever activated accidentally (e.g., through a direct DB edit or future code path), Draft campaigns will still show all modules as CLOSE
- The guard also protects campaigns that are manually set back to Draft status

**Alternatives considered:**
- *Only fix duplication*: Simpler but fragile — any other code path that sets `isActive = true` on a Draft campaign's phase config would re-introduce the bug
- *Only add Draft guard*: Would fix the display but leave `isActive = true` on phase configs, potentially causing issues if other code reads this flag directly
- *Clear dates during duplication*: Rejected — dates should be preserved as a template for when the PM is ready to activate

### Decision 2: Campaign status check mechanism

The campaign status is stored on the `Session` entity (related to Campaign). Check `$campaign->getSession()->getStatus()->getName()` for `'draft'` (or the equivalent status entity value). Need to verify the exact Draft status name used in `SessionStatus`.

**Approach**: Look up how other parts of the codebase check for draft status and follow the same pattern.

### Decision 3: Apply fix to both admin and student mappers

Both `CampaignService` (admin-facing module list) and `CampaignToModuleDetailDtoMapper` (student-facing module details) compute `is_active` from date ranges. Both need the Draft guard. The `SimulationDashboardService` also references `is_active` but delegates to `CampaignService` data — verify if it needs a direct fix.

## Risks / Trade-offs

- **[Risk] Incorrect Draft status name** → Verify the exact `SessionStatus` name/code for Draft before implementing. Check `SessionStatus` entity or existing conditional checks in the codebase.
- **[Risk] Other code paths reading `CampaignPhaseConfig.isActive` directly** → The duplication fix (setting to false) mitigates this. A codebase search for `.isActive()` on phase configs should confirm no other consumers are affected.
- **[Risk] Campaigns set back to Draft after being Open** → The Draft guard will force all modules to CLOSE. This is the desired behavior per acceptance criteria, but verify that no workflow depends on preserving module status when reverting to Draft.
- **[Trade-off] Two-layer fix vs single fix** → Slightly more code, but significantly more robust. The guard ensures correctness even if duplication logic is modified in the future.
