# PM Dashboard — Fix Seat Column Split Format

## Problem
Jira: [DPBFAD-726](https://insead.atlassian.net/browse/DPBFAD-726)

The "Seat Available" column in the PM's bidding round course table displays the split format `XX (XX)` whenever `total_all_seats !== total_seats`. This is incorrect — the split format should only appear when a course is genuinely shared/split across multiple programme-promotions, not merely when the values differ (e.g., due to an `AdjustmentCourse` seat override on a non-shared course).

## Changes

### 1. `src/components/dashboard/process/bidding-round/course-table-bidding-round.tsx` — Fix Seat Column Condition
- Replaced the condition `row.total_all_seats && row.total_all_seats !== row.total_seats` with `row.is_shared`.
- When `is_shared` is `true`: displays `{total_seats} ({total_all_seats})` — e.g., `18 (48)`.
- When `is_shared` is `false` or undefined: displays only `{total_seats}` — e.g., `40`.

### 2. `src/src/campaign-management/campaign.types.ts` — Add `is_shared` to Type
- Added `is_shared?: boolean` field to the `CourseListBidding` type to match the new API response field.

## Depends On
- **bidding-api PR**: Adds the `is_shared` boolean to the bidding round course detail API response. This admin PR should be deployed together with (or after) the API PR. If deployed before, the old behavior continues (no `is_shared` field means the split format is never shown).

## Verification Steps
1. Open PM dashboard → Bidding Round detail for an active campaign.
2. Verify non-shared courses show only `XX` in the seat column.
3. Verify shared courses (split across multiple promotions) show `XX (XX)` format.
4. Verify courses with adjusted seats but only one promotion do **not** show the split format.
