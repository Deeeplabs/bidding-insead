## Why

The Student Portal (`bidding-web`) incorrectly displays bidding round closing and opening times, causing an 8-hour shift for users in the SGT (GMT+8) timezone. For instance, a round scheduled to close at 1:20 AM SGT is displayed as closing at 9:20 AM SGT.

This issue is caused by a two-fold bug in the processing pipeline:
1. **Backend "Fake UTC" Issue**: `DateHelper::toIso` in `bidding-api` was formatting dates as ISO strings with a `Z` suffix but *without* converting them to UTC first. This meant local server times (e.g., Paris or SGT) were being incorrectly presented as true UTC.
2. **Double-Parsing Bug (Frontend)**: Components (`CollapsePanelExtra` and `HeaderSection`) in `bidding-web` were stripping timezone context during intermediate formatting, then re-parsing these local strings as UTC, adding the offset a second time.

## What Changes

- **Fix `DateHelper.php` (Backend)**: Refactor to convert all `DateTimeInterface` objects to UTC before applying the `Z` suffix.
- **Fix Student API (Backend)**: Standardize date fields in student-facing controllers to use `DateHelper::toIso()` instead of manual formatting.
- **Fix `CollapsePanelExtra.tsx` (Frontend)**: Refactor to pass `Date` objects directly to phase utilities, avoiding timezone-stripping formatting.
- **Fix `HeaderSection.tsx` (Frontend)**: Similarly refactor to maintain `Date` context throughout the display pipeline.

## Capabilities

### Modified Capabilities

- **Accurate Bidding Round Status**: Students now see the correct local times for bidding round starts and ends, aligned with the PM's configuration across all server regions.

## Impact

- **bidding-api**: `src/Helper/DateHelper.php` — Fix UTC conversion logic and add robust parser.
- **bidding-api**: `src/Domain/Campaign/Campaign/CampaignService.php` — Ensure UTC interpretation on save.
- **bidding-api**: `src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToModuleDetailDtoMapper.php` — Use robust parser for mapping.
- **bidding-web**: `src/features/bidding/components/shared/bidding-card/CollapsePanelExtra.tsx` — Refactor to use `Date` objects.
- **bidding-web**: `src/features/bidding/components/bid-submission/HeaderSection.tsx` — Refactor to use `Date` objects.
- **Affected roles**: Students and Administrators.
- **No migration required**: No database schema changes.
- **No regression risk**: Fixing the parsing logic restores intended behavior for all timezones.
