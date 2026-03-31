## Current Implementation Root Cause

In `bidding-web`, the `CollapsePanelExtra` and `HeaderSection` components follow a data handling pattern that introduces a double-parsing error:
1. `parseUtc(date_from_api)` -> Returns a `Date` object in UTC.
2. `useDate().format(date, 'yyyy-MM-dd HH:mm:ss')` -> Returns a local time string *without* timezone info (e.g., `"2026-03-27 01:20:00"` for SGT).
3. This local string is passed to `getPhaseStatus` and `getPhaseDisplayText`.
4. These utilities call `parseUtc()` on the input string.
5. `parseUtc()` interprets strings without timezone info as UTC and appends 'Z' -> `2026-03-27T01:20:00Z`.
6. Final rendering using `useDate().format()` adds the timezone offset *again* (e.g., +8 for SGT), resulting in `09:20:00 SGT` instead of the original `01:20:00 SGT`.

## Proposed Design

The solution is to maintain the date as a `Date` object (or an ISO string with timezone context) throughout the processing pipeline and only format it for final display in the UI.

### Component Refactor: `CollapsePanelExtra.tsx`
- **Location**: `src/features/bidding/components/shared/bidding-card/CollapsePanelExtra.tsx`
- **Change**: Replace premature string formatting with direct `Date` objects returned by `parseUtc`.
```typescript
// From:
const parsedStartDate = format(parseUtc(startDate), 'yyyy-MM-dd HH:mm:ss');
const parsedEndDate = format(parseUtc(endDate), 'yyyy-MM-dd HH:mm:ss');

// To:
const parsedStartDate = parseUtc(startDate);
const parsedEndDate = parseUtc(endDate);
```
- Since `getPhaseStatus` and `getPhaseDisplayText` already call `parseUtc` internally (which is idempotent for `Date` objects), this fix is backward-compatible with the utility's signature.

### Component Refactor: `HeaderSection.tsx`
- **Location**: `src/features/bidding/components/bid-submission/HeaderSection.tsx`
- **Change**: Similarly refactor lines 58-59 to avoid string loss of timezone context.
```typescript
// From:
const parsedStartDate = format(parseUtc(startDate), 'yyyy-MM-dd HH:mm:ss');
const parsedEndDate = format(parseUtc(endDate), 'yyyy-MM-dd HH:mm:ss');

// To:
const parsedStartDate = parseUtc(startDate);
const parsedEndDate = parseUtc(endDate);
```

### Affected Files Summary
| File Path | Impact |
|-----------|--------|
| `src/features/bidding/components/shared/bidding-card/CollapsePanelExtra.tsx` | Refactor date parsing logic |
| `src/features/bidding/components/bid-submission/HeaderSection.tsx` | Refactor date parsing logic |

## Risk Assessment & Safety
- **No API impact**: This is a purely frontend display fix.
- **No data risk**: Business logic calculations (which should use UTC) are not affected, only UI display strings.
- **Regression**: Check if other components use this intermediate formatting pattern. `getPhaseStatus` and `getPhaseDisplayText` are specifically affected because they "re-parse" the input.
