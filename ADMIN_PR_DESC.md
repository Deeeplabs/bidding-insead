# Fix student pagination not working properly in Create Campaign page

## Problem
**Jira**: https://insead.atlassian.net/browse/DPBFAD-747

Student pagination in the Create Campaign page is broken. The `StudentList` component's `queryPayload` state management has multiple bugs that cause pagination to malfunction when filters, sorting, or exclude/include actions are applied.

**Symptoms:**
- After applying or removing student filters (exclude, include, campus, credit range), the table shows empty results because the user is stranded on a page that no longer exists
- Clearing filters does not reset the page
- Removed filter parameters (e.g., `campus_ids`, `credits_min`) persist in the API request, returning incorrectly filtered data and wrong pagination totals
- Sort parameters can revert to stale values when other dependencies trigger the `useEffect`
- After changing rows per page (pageSize), the pagination controls and actual data count don't match

## Root Cause

Five interacting bugs across `student-list-table.tsx` and `student-setting-table.tsx`:

1. **Page not resetting on filter changes** — The `useEffect` that rebuilds `queryPayload` used `page: prev?.page || 1`, preserving the current page when `studentFilters`/`includedIds`/`excludedIds` change. If the user is on page 3 and excludes a student (reducing total records), page 3 may no longer exist → empty results.

2. **Stale filter keys not cleared** — The `useEffect` built the payload via `...prev` spread followed by conditional additions. When a filter was removed (e.g., `campusFilter` becomes `[]`), the conditional added nothing — but old `campus_ids` from `...prev` persisted. The API received stale filter params.

3. **`currentSort` missing from useEffect deps** — `currentSort` was used in the effect body but not listed in the dependency array. When another dep triggered the effect, `currentSort` was read from a stale closure, potentially reverting sort params.

4. **`handleClear` stale closure and no page reset** — Used `...queryPayload` from the closure instead of functional `(prev) => ...` pattern. Did not reset `page` to 1 or clear filter keys.

5. **`StyledPagination` pageSize hardcoded** — In `student-setting-table.tsx`, `pageSize={PAGINATION_LIMIT}` was hardcoded to 10. When the user changed rows per page via Ant Design's size changer, `handleChangePage` correctly updated `queryPayload.limit` (e.g., to 20) and the API returned 20 items, but the pagination component still rendered with `pageSize=10`, causing wrong page count and mismatched display.

## Solution

### 1. Clean payload rebuild in useEffect

Replaced the `...prev` spread with a clean object construction. Page always resets to 1. Stale filter keys are eliminated since we no longer inherit from `prev`. Only `search` and `limit` are explicitly preserved from previous state:

```tsx
setQueryPayload((prev) => ({
  page: 1,
  limit: prev?.limit || PAGINATION_LIMIT,
  programme_ids: [selectedProgramme.id],
  sort: currentSort?.sort || 'first_name',
  order: currentSort?.order || 'asc',
  ...(promotionId && { promotion_id: Number(promotionId) }),
  ...(campusFilter?.length ? { campus_ids: campusFilter } : {}),
  ...(typeFilter?.length ? { student_types: typeFilter } : {}),
  ...creditParams,
  ...capitalParams,
  include_student: includedIds.length > 0 ? includedIds : undefined,
  exclude_student: excludedIds.length > 0 ? excludedIds : undefined,
  ...(prev?.search ? { search: prev.search } : {}),
}));
```

### 2. Added `currentSort` to useEffect dependency array

Fixes the stale closure. Sort changes now correctly trigger a payload rebuild.

### 3. Fixed `handleClear` with functional update

Uses `(prev) => ...` pattern, resets page to 1, preserves only `promotion_id`, `sort`, and `order` from previous state — all filter keys are cleared.

### 4. Dynamic `pageSize` in `StudentsSettingTable`

Added a `pageSize` prop to `StudentsSettingTable` and pass the current `queryPayload.limit` from `StudentList`. The `StyledPagination` now uses this dynamic value instead of the hardcoded `PAGINATION_LIMIT`. Also added `showSizeChanger` to explicitly enable the rows-per-page dropdown.

## Files Changed

| File | Change |
|------|--------|
| `src/components/campaign/student/student-list-table.tsx` | Fixed `useEffect` payload rebuild (clean object, page reset, search preserved); added `currentSort` to deps; fixed `handleClear` with functional update and page reset; pass dynamic `pageSize` to `StudentsSettingTable` |
| `src/components/settings/student-setting-table.tsx` | Added `pageSize` prop; use dynamic `pageSize` in `StyledPagination` instead of hardcoded `PAGINATION_LIMIT`; added `showSizeChanger` |

## Impact

- **Fixes empty page results** — Page resets to 1 whenever filters change
- **Fixes stale filters** — Removed filter params no longer persist in API requests
- **Fixes sort reversion** — Sort state stays consistent across filter changes
- **Fixes clear behavior** — Clear properly resets page and removes all filters
- **Fixes rows-per-page mismatch** — Pagination display now matches the actual page size after changing rows per page
- **No breaking changes** — No API contracts changed, no database changes

## Testing

Manual verification needed:
- Navigate to Create Campaign → student list → go to page 2+, apply a filter → verify page resets to 1 with correct data
- Apply a campus filter, then remove it → verify `campus_ids` is no longer in the API request (Network tab)
- Exclude a student while on page 2+ → verify page resets to 1
- Sort by a column, then apply a filter → verify sort order is maintained
- Click clear after applying filters on page 3 → verify page resets to 1 and filters are removed
- Basic pagination (no filters) → click pages 1, 2, 3 → verify correct data loads
- Change rows per page (e.g., 10 → 20) → verify table shows 20 rows and pagination reflects the new page size
