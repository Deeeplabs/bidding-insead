## Context

The `StudentList` component in `student-list-table.tsx` manages a `queryPayload` state object that drives the `useListStudents` React Query hook. Pagination, filtering, sorting, and student include/exclude parameters are all merged into this single state object.

**Component Architecture:**

- `StudentList` (`student-list-table.tsx`) — owns `queryPayload` state, passes pagination data and `handleChangePage` to child
- `StudentsSettingTable` (`student-setting-table.tsx`) — renders table rows and `StyledPagination` (Ant Design Pagination wrapper), calls `onChange(page, pageSize)` on page click
- `useListStudents` (`use-list-student.ts`) — React Query hook that fetches `GET /students` with `queryPayload` as params
- `useCampaignStore` (Zustand) — global store holding `studentFilters` array, updated via `updateStudentFilters`

**The queryPayload lifecycle:**

1. **Initialization:** A large `useEffect` (lines 318-397) fires when `selectedProgramme`, `selectedPromotion`, `studentFilters`, `includedIds`, or `excludedIds` change. It builds the full payload from these sources.
2. **Pagination:** `handleChangePage(page, pageSize)` updates `page` and `limit` in payload.
3. **Search/Filter bar:** `handleSearch(params)` resets page to 1 and merges filter params.
4. **Clear:** `handleClear()` resets limit and programme_ids.

**Four bugs identified:**

### Bug 1: Page not reset on filter-dependency changes (PRIMARY)

The `useEffect` at lines 373-387 uses:
```tsx
page: prev?.page || 1,
```
This preserves the current page when `studentFilters`/`includedIds`/`excludedIds` change. If the user is on page 3 and excludes a student (reducing total records), page 3 may no longer exist, resulting in empty results from the API.

### Bug 2: Stale filter keys not cleared

The `useEffect` builds the payload via `...prev` spread followed by conditional additions:
```tsx
...(campusFilter?.length && { campus_ids: campusFilter }),
...(typeFilter?.length && { student_types: typeFilter }),
```
When a filter is removed (e.g., `campusFilter` becomes `[]`), the conditional evaluates to falsy and adds nothing — but the old `campus_ids` from `...prev` persists. The API receives stale filter params, returning wrong data and wrong pagination totals.

### Bug 3: `currentSort` missing from useEffect deps

Lines 378-379 reference `currentSort` in the effect body:
```tsx
sort: currentSort?.sort || 'first_name',
order: currentSort?.order || 'asc',
```
But `currentSort` is not in the dependency array (lines 389-397). When another dependency triggers the effect, `currentSort` is read from a stale closure, potentially reverting sort params.

### Bug 4: `handleClear` stale closure and no page reset

```tsx
const handleClear = () => {
    setQueryPayload({
      ...queryPayload,  // stale closure — not functional update
      limit: PAGINATION_LIMIT,
      programme_ids: selectedProgramme?.id ? [selectedProgramme.id] : [],
    });
  };
```
Uses `queryPayload` from the closure instead of `(prev) => ...` pattern. Also does not reset `page` to 1, does not clear search/filter keys.

## Goals / Non-Goals

**Goals:**
- Reset page to 1 whenever filter-related dependencies change in the useEffect
- Explicitly clear stale filter keys when rebuilding payload in the useEffect
- Fix stale `currentSort` closure by adding it to dependency array
- Fix `handleClear` to use functional update and reset page

**Non-Goals:**
- Refactoring the overall state management approach (single payload object vs separate state)
- Changing the `StudentsSettingTable` pagination component
- Modifying the `useListStudents` hook or API

## Decisions

**Decision: Reset page to 1 in the useEffect when filter deps change**

Change `page: prev?.page || 1` to `page: 1` in the useEffect. This ensures that whenever `studentFilters`, `includedIds`, or `excludedIds` change, the user is returned to page 1. The `handleChangePage` function handles user-initiated page navigation separately and is unaffected.

**Decision: Build a clean payload object instead of spreading prev**

Instead of `{...prev, ...(conditional)}`, construct the payload explicitly, setting filter keys to `undefined` when not applicable. This ensures removed filters don't persist:

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

Key differences from current code:
- `page: 1` — always reset
- No `...prev` spread — clean object, no stale keys
- Preserves `search` from prev explicitly (since search is managed separately via `handleSearch`)

**Decision: Add `currentSort` to useEffect dependency array**

This fixes the stale closure. When sort changes, the effect re-runs and keeps the payload consistent.

**Decision: Fix `handleClear` with functional update and page reset**

```tsx
const handleClear = () => {
    setQueryPayload((prev) => ({
      page: 1,
      limit: PAGINATION_LIMIT,
      programme_ids: selectedProgramme?.id ? [selectedProgramme.id] : [],
    }));
  };
```

## Risks / Trade-offs

**[Risk] Resetting page to 1 on every useEffect trigger may feel jarring**
The useEffect fires when `studentFilters` change. If the user applies a filter while on page 3, they jump to page 1. This is standard UX behavior for filtered lists and is preferable to showing empty results.

**[Risk] Removing `...prev` spread may lose unknown future payload keys**
By building a clean object, any payload keys added outside the useEffect (e.g., `search`) must be explicitly preserved. The design handles `search` by conditionally copying it from `prev`. If new keys are added in the future, they must be accounted for.

**[Risk] Adding `currentSort` to deps causes extra useEffect runs on sort change**
When the user sorts, `currentSort` changes, triggering the useEffect which resets page to 1. This is actually desirable — when sort changes, page 1 of the new sort order should be shown. The `handleSort` function already sets `page: 1`, so this is consistent.

## Migration Plan

1. **Code change** — Fix `useEffect` in `student-list-table.tsx` (lines 318-397)
2. **Code change** — Fix `handleClear` in `student-list-table.tsx` (lines 269-275)
3. **No database migration required**
4. **Deployment:** Standard Next.js build and deploy
5. **Rollback:** Simple code revert if issues arise
6. **Testing:** Manual verification of pagination across filter/sort/clear scenarios
