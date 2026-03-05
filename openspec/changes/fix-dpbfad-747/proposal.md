## Why

Student pagination in the Create Campaign page (`student-list-table.tsx`) is broken due to multiple interacting issues in how `queryPayload` state is managed. The core problem is a large `useEffect` (lines 318-397) that rebuilds the query payload whenever `studentFilters`, `includedIds`, `excludedIds`, or promotion/programme IDs change — but it fails to reset the page to 1 when filter-related dependencies change, and it fails to clear stale filter keys from the previous payload.

**Symptoms:**
- After applying/removing student filters (exclude, include, campus, credit range), the table may show empty results because the user is stranded on a page that no longer exists
- Clearing filters does not reset the page
- Removed filter parameters (e.g., `campus_ids`, `credits_min`) persist in the API request, returning incorrectly filtered data and wrong pagination totals
- Sort parameters can revert to stale values when other dependencies trigger the `useEffect`

## What Changes

- Fix the `useEffect` in `student-list-table.tsx` to reset page to 1 when filter-related deps change
- Fix the `useEffect` to explicitly clear stale filter keys (`campus_ids`, `student_types`, `credits_min`, `credits_max`, `capital_min`, `capital_max`) instead of only conditionally adding them
- Add `currentSort` to the `useEffect` dependency array to fix stale sort closure
- Fix `handleClear` to use functional state update and reset page to 1

## Capabilities

### New Capabilities
- None — this is a bug fix

### Modified Capabilities
- `student-list-pagination`: Fix pagination state management so page resets properly on filter changes and stale filter params are cleared

## Impact

**Affected Code:**
- `bidding-admin/src/components/campaign/student/student-list-table.tsx` — `useEffect` (lines 318-397), `handleClear` (lines 269-275)

**Testing:**
- Verify pagination resets to page 1 when applying/removing filters (campus, credit range, student type, exclude, include)
- Verify clearing filters resets to page 1 and removes filter params from API request
- Verify sorting persists correctly across filter changes
- Verify page navigation still works normally (clicking page 2, 3, etc.)
- Verify the Create Campaign page student list works end-to-end

**No Breaking Changes:**
- This is a bug fix that corrects state management behavior
- No API contracts are changed
- No database changes required
