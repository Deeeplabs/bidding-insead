## Why

The Student Portal (`bidding-web`) incorrectly displays bidding round closing and opening times, causing an 8-hour shift for users in the SGT (GMT+8) timezone. In addition, durations for bidding rounds are frequently off by 1 or 2 hours across daylight saving time shifts, causing countdowns to display "1d 22h" instead of the expected "1d 23h".

This issue is caused by a three-fold bug in the processing pipeline:
1. **Database Naive UTC Drift**: The backend stores dates in MySQL `DATETIME` columns without timezone information. Doctrine reads these and instantiates `DateTime` objects assuming the server's local timezone (Europe/Paris).
2. **Incorrect UTC Conversion (`DateHelper::toIso`)**: `DateHelper::toIso` was blindly appending a `Z` suffix. While the previous attempt to fix this converted the Paris-timezone object to UTC, it actually modified the hour, shifting `17:00 UTC` (which Doctrine labeled as `17:00 Paris`) to `16:00 UTC`, creating the 1-2 hour DST discrepancy ("1d 23h" issue).
3. **Double-Parsing Bug (Frontend)**: Components (`CollapsePanelExtra` and `HeaderSection`) in `bidding-web` were stripping timezone context during intermediate formatting, then re-parsing these local strings as UTC, adding the offset a second time.

## What Changes

- **Fix `DateHelper.php` (Backend)**: Refactor `toIso()` to reinterpret the naive datetime string from Doctrine as UTC *without* shifting the hours. Add a `parse()` method that guarantees timezone-aware UTC datetime instantiation before saving to the database.
- **Fix Student API (Backend)**: Standardize date fields in student-facing controllers to use `DateHelper::toIso()` instead of manual formatting.
- **Fix `CollapsePanelExtra.tsx` (Frontend)**: Refactor to pass `Date` objects directly to phase utilities, avoiding timezone-stripping formatting.
- **Fix `HeaderSection.tsx` (Frontend)**: Similarly refactor to maintain `Date` context throughout the display pipeline.

## Capabilities

### Modified Capabilities

- **Accurate Bidding Round Status**: Students now see the correct local times for bidding round starts and ends, aligned precisely with the PM's configuration across all server regions, ensuring countdowns strictly match the expected configured duration (eliminating the "1d 23h" DST gap).

## Impact

- **bidding-api**: `src/Helper/DateHelper.php` — Fix UTC re-interpretation logic and add robust parser.
- **bidding-api**: `src/Domain/Campaign/Campaign/CampaignService.php` — Ensure UTC interpretation on save.
- **bidding-api**: `src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToModuleDetailDtoMapper.php` — Use robust parser for mapping.
- **bidding-web**: `src/features/bidding/components/shared/bidding-card/CollapsePanelExtra.tsx` — Refactor to use `Date` objects.
- **bidding-web**: `src/features/bidding/components/bid-submission/HeaderSection.tsx` — Refactor to use `Date` objects.
- **Affected roles**: Students and Administrators.
- **No migration required**: No database schema changes.
- **No regression risk**: Fixing the parsing logic restores intended behavior for all timezones and eliminates DST duration shifting.
