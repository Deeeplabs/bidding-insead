## Why

The Student Portal (`bidding-web`) incorrectly displays bidding round closing and opening times, causing an 8-hour shift for users in the SGT (GMT+8) timezone. For instance, a round scheduled to close at 1:20 AM SGT is displayed as closing at 9:20 AM SGT.

This issue is caused by a double-parsing bug in the frontend components:
1. Components (`CollapsePanelExtra` and `HeaderSection`) parse the UTC date string from the API into a `Date` object using `parseUtc`.
2. They then format this `Date` object into a local time string (e.g., `"2026-03-27 09:20:00"`) *without* timezone information using `useDate().format`.
3. This local string is then passed to phase display utilities (`getPhaseStatus`, `getPhaseDisplayText`) that call `parseUtc` again.
4. `parseUtc` interprets strings without timezone info as UTC, incorrectly shifting the time by the user's timezone offset a second time during formatting.

## What Changes

- **Fix `CollapsePanelExtra.tsx`**: Refactor the component to pass the `Date` objects returned by `parseUtc` directly to the phase status and display utilities, rather than intermediate formatted strings.
- **Fix `HeaderSection.tsx`**: Apply the same refactor to avoid premature string conversion that loses timezone context.
- **Maintain UTC Source of Truth**: Ensure all date-dependent calculations in `bidding-web` leverage the `Date` objects directly until the final rendering step.

## Capabilities

### New Capabilities

_(none — this is a fix for existing time display logic)_

### Modified Capabilities

- **Accurate Bidding Round Status**: Students now see the correct local times for bidding round starts and ends, aligned with the PM's configuration and the backend source of truth.

## Impact

- **bidding-web**: `src/features/bidding/components/shared/bidding-card/CollapsePanelExtra.tsx` — Refactor to use `Date` objects for `parsedStartDate` and `parsedEndDate` to avoid double-parsing shifts.
- **bidding-web**: `src/features/bidding/components/bid-submission/HeaderSection.tsx` — Refactor to use `Date` objects for `parsedStartDate` and `parsedEndDate`.
- **Affected roles**: Students.
- **No API changes required**: Backend already returns correct UTC date-time values.
- **No migration required**: No database schema changes.
- **No regression risk**: Fixing the parsing logic restores intended behavior for all timezones.
