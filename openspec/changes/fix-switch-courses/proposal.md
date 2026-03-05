## Why

In the PM admin panel under Switch → All Courses, pagination is broken when changing rows per page to 100. The total page count reports more pages than actually contain data — navigating to the last pages (e.g., page 69) shows "No records found" until going back to around page 66.

**Root cause**: The `ClassesRepository::searchQueryPaginated()` method uses `LEFT JOIN` on `classPromotions` and `promotion`, with a `GROUP BY` that includes `cp.id` and `p.id`. When a class has multiple promotion associations, the same class appears as multiple rows (one per promotion). Doctrine's `Paginator->count()` counts these inflated rows, producing a `totalRecords` and `totalPages` higher than the actual number of distinct classes. When the user navigates to pages beyond the real data range, the query returns empty results.

The `fetchJoinCollection: true` flag in the Paginator helps deduplicate the actual results returned on each page, but the **count query** still counts the inflated row set, leading to phantom pages at the end.

## What Changes

Fix the pagination count in the `searchQueryPaginated` method to accurately count distinct classes rather than inflated joined rows. This ensures the total page count matches the actual number of pages with data.

## Capabilities

### New Capabilities
_(none)_

### Modified Capabilities
- `flex-switch-list-course`: Fix pagination total count to use `COUNT(DISTINCT cl.id)` instead of counting inflated joined rows, ensuring accurate page navigation in the All Courses list.

## Impact

- **API (`bidding-api`)**: `ClassesRepository::searchQueryPaginated()` — the pagination count logic needs to produce accurate totals that account for the `LEFT JOIN` row inflation.
- **Affected endpoint**: `GET /flex-switch/list-course` (used by PM → Switch → All Courses tab)
- **Affected files**:
  - `src/Repository/ClassesRepository.php` — `searchQueryPaginated()` method
  - `src/Repository/PaginatedResult.php` — potentially adjust how count is performed
- **No migration required**: This is a query logic fix, no schema changes needed.
- **No breaking API changes**: The response shape stays the same; only the `pagination.total` and `pagination.total_pages` values will become accurate.
- **No frontend changes needed**: The Ant Design `<Pagination>` component in `bidding-admin` already correctly uses the API's pagination response.
