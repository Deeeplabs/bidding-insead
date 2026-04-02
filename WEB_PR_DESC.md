# Static Seat Display for Students

## Problem
Jira: [DPBFAD-726](https://insead.atlassian.net/browse/DPBFAD-726)

During both the Bidding Round and Add/Drop phases, the course selector displays a dynamic seat chip showing `{available_seats}/{total_seats} seats` with green/red coloring based on availability. This reveals real-time seat consumption to students, causing confusion and anxiety. The PM requires that seat capacity be displayed as a **static** total — students should only see how many seats the course offers, never remaining seats or enrollment counts.

## Changes

### 1. `CourseOptionContent.tsx` — Replace Dynamic Seat Chip with Static Display
- Changed the seat chip from `{data.available_seats}/{data.total_seats} seats` to `{data.total_seats} seats`.
- Removed conditional green/red coloring based on `data.available_seats > 0`.
- Applied a single neutral chip style: `!border !bg-white !text-gray-700 !border-gray-300`.
- The chip now renders when `data.total_seats` is defined (no longer requires `available_seats`).
- This single change applies to both Bidding Round and Add/Drop since both phases use the same `CourseOptionContent` component via `CourseSelector.tsx`.

### 2. Verification — No Other Student-Facing Components Render Dynamic Seats
- Confirmed that `available_seats` and `enrolled_count` exist only in type definitions (`bidding.type.ts`, `add-drop.type.ts`, etc.) and data mapping (`AddDropPage.tsx`) — but **no UI component renders them** to the student.
- The API still returns these fields for backward compatibility; only the UI rendering was changed.

## Impact
- **Static Seat Display**: Students see only `XX seats` (total configured capacity) — no remaining/available counts, no green/red coloring.
- **Consistent Across Phases**: Applies to both Bidding Round and Add/Drop course selectors.
- **No Breaking Changes**: API response fields unchanged. Only the UI rendering was updated.
- **Backend Enforcement Preserved**: Real-time seat validation continues server-side. Students attempting to enroll in a full course receive a clear error at submission time.

## Verification Steps
1. During an active Bidding Round, open the course selector — verify each course shows only `XX seats` with neutral styling.
2. During an active Add/Drop, open the course selector — verify the same static display.
3. Attempt to enroll in a full course — verify a clear error appears (WAITLISTED status or "Course is full").
4. Confirm no course option shows remaining seat counts, enrolled counts, or availability-based coloring.
