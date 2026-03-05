# Fix Switch All Courses Pagination Count

## Problem

In the PM admin panel under **Switch → All Courses**, the pagination reports more pages than actually contain data. When changing rows per page to 100 and navigating to the last pages (e.g., page 69), "No records found" is displayed until going back several pages (around page 66).

**Root cause:** The `ClassesRepository::searchQueryPaginated()` method uses `LEFT JOIN` on `classPromotions` and `promotion`, with a `GROUP BY` that includes `cp.id` and `p.id`. When a class has multiple promotion associations (multiple rows in `class_promotions`), each class-promotion pair becomes a separate group. Doctrine's `Paginator->count()` wraps this grouped query as `SELECT COUNT(*) FROM (SELECT ... GROUP BY ...)`, counting the inflated number of groups instead of the actual distinct classes — producing a `totalRecords` and `totalPages` that are too high.

## Solution

Introduced a separate count query using `COUNT(DISTINCT cl.id)` that runs before pagination is applied, bypassing Doctrine's Paginator count which miscounts with `GROUP BY` + JOINs. Extended `PaginatedResult` to accept an optional pre-computed total override.

### Changes Made

**Modified Files:**

1. **`src/Repository/PaginatedResult.php`**
   - Added optional `?int $totalOverride = null` parameter to the constructor
   - When `$totalOverride` is provided, uses it instead of `Paginator->count()` for `totalRecords` and `totalPages`
   - Existing callers that don't pass `$totalOverride` are unaffected (backward compatible)

2. **`src/Repository/ClassesRepository.php`**
   - Modified `searchQueryPaginated()` to compute an accurate count before applying pagination
   - After all filters, JOINs, WHERE clauses, and GROUP BY are applied, clones the query builder
   - Resets the clone's SELECT to `COUNT(DISTINCT cl.id)`, removes GROUP BY and ORDER BY
   - Executes the count query to get the true number of distinct classes
   - Passes the result as `totalOverride` to `PaginatedResult`

## Impact

- **Affected endpoint:** `GET /flex-switch/list-course` (PM → Switch → All Courses tab)
- **No breaking changes:** API response shape is unchanged — only `pagination.total` and `pagination.total_pages` values become accurate
- **No migration required:** Query logic fix only, no schema changes
- **Backward compatible:** `PaginatedResult` default behavior unchanged for all other callers
- **Minimal performance impact:** One additional lightweight `COUNT(DISTINCT)` query per request

## Testing

- [ ] Verified pagination total matches actual distinct class count via API
- [ ] Verified last page (as indicated by `total_pages`) returns data in admin UI
- [ ] Verified no phantom empty pages beyond the last data page
- [ ] Verified search/filter combinations still produce accurate pagination
- [ ] Verified export (no pagination) is unaffected
- [ ] Verified other paginated endpoints using `searchQueryPaginated` are unaffected
