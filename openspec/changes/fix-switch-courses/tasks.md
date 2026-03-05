## 1. Backend Fix — PaginatedResult Override Support

- [x] 1.1 **Add `$totalOverride` parameter to `PaginatedResult`**
  - File: `bidding-api/src/Repository/PaginatedResult.php`
  - Add an optional `?int $totalOverride = null` parameter to the constructor
  - When `$totalOverride` is not null, use it instead of `$this->paginator->count()` for `totalRecords` and `totalPages`
  - Existing callers that don't pass `$totalOverride` continue to work as before

## 2. Backend Fix — Accurate Count in searchQueryPaginated

- [x] 2.1 **Add separate count query in `ClassesRepository::searchQueryPaginated()`**
  - File: `bidding-api/src/Repository/ClassesRepository.php`
  - Method: `searchQueryPaginated()`
  - After building the main query with all filters, JOINs, and WHERE clauses (but before applying pagination via `FilterableQueryProvider`):
    1. Clone the query builder
    2. Reset the clone's SELECT to `SELECT COUNT(DISTINCT cl.id)`
    3. Remove the `GROUP BY` from the clone
    4. Execute the count query to get the accurate total
  - Pass the computed count as `$totalOverride` when constructing `PaginatedResult`
  - This fixes both the standard GROUP BY mode and the `_campaign_group_mode` path

## 3. Manual Verification

- [ ] 3.1 **Verify pagination accuracy via API**
  - Start the bidding-api dev server
  - Call `GET /flex-switch/list-course?page=1&limit=100&programmeId=<valid_id>` and note the `pagination.total` and `pagination.total_pages`
  - Navigate to the last page (page = `total_pages`) — verify it returns data
  - Navigate to page `total_pages + 1` — verify it returns empty `items`

- [ ] 3.2 **Verify in admin UI**
  - Login as PM in `bidding-admin`
  - Go to Switch → All Courses
  - Change rows per page to 100
  - Navigate to the last page — verify it shows records
  - Confirm no phantom empty pages exist beyond the last data page
  - Test with search/filter applied — verify pagination still accurate

- [ ] 3.3 **Regression check**
  - Verify the standard 10-per-page pagination still works correctly
  - Verify export (no pagination) still returns all records
  - Verify other paginated endpoints using `searchQueryPaginated` (e.g., class configuration) are unaffected
