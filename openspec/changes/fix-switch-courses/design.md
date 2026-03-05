## Context

The PM admin panel's Switch → All Courses tab uses `GET /flex-switch/list-course` to display a paginated list of classes. The request flows through:

1. **Controller**: `FlexSwitchController::listCourse()` → builds filter groups, calls service
2. **Service**: `FlexSwitchService::listCourse()` → calls `ClassService::getList()` with a `PaginationList`
3. **Repository**: `ClassesRepository::searchQueryPaginated()` → builds a DQL query with JOINs
4. **Pagination**: `PaginatedResult` → uses Doctrine's `Paginator->count()` for totals

**The bug**: `searchQueryPaginated()` uses `LEFT JOIN` on `classPromotions` (`cp`) and `promotion` (`p`), then `GROUP BY cl.id, c.id, s.id, ct.id, cam.id, stat.id, cp.id, p.id`. When a class has multiple promotion associations (multiple `class_promotions` rows), the GROUP BY produces one row per class+promotion pair. Doctrine's `Paginator->count()` wraps the grouped query as `SELECT COUNT(*) FROM (SELECT ... GROUP BY ...)`, counting the number of groups — not the number of distinct classes.

This means `totalRecords` is inflated, making `totalPages` too high. Navigating past the actual data range returns empty results ("No records found").

**Frontend**: `ListCourseFlex` component (`bidding-admin/src/components/flex-switch/course-flex-list.tsx`) uses Ant Design's `<Pagination>` which relies on `pagination.total` from the API response. The frontend is correctly implemented — the bug is purely in the backend count.

## Goals / Non-Goals

**Goals:**
- Fix the pagination count in `searchQueryPaginated()` so that `totalRecords` reflects the actual number of distinct class rows returned by the query
- Ensure the fix works with all existing filter combinations (search, credit, seat, campus, mode)
- Ensure the fix works for both the standard mode and the `_campaign_group_mode` / `_unique_by_course` special modes

**Non-Goals:**
- Changing the API response shape
- Changing the frontend pagination behavior
- Fixing pagination in other endpoints (e.g., `listClassConfiguration`)
- Adding new filters or functionality

## Decisions

### Decision 1: Use a custom count query instead of relying on Doctrine Paginator's built-in count

**Rationale**: Doctrine's `Paginator->count()` doesn't handle `GROUP BY` well — it wraps the query in `SELECT COUNT(*) FROM (...)` which counts group rows, not distinct entities. Rather than trying to work around Doctrine's behavior, we'll compute the count separately.

**Approach**: Modify `searchQueryPaginated()` to:
1. Build a separate count query (clone the base query before pagination is applied)
2. Use `SELECT COUNT(DISTINCT cl.id)` to get the true number of distinct classes
3. Remove the `GROUP BY` from the count query (since we only need DISTINCT count)
4. Pass the pre-computed count to `PaginatedResult` so it doesn't use `Paginator->count()`

### Decision 2: Extend PaginatedResult to accept an optional pre-computed total

**Rationale**: `PaginatedResult` currently always calls `$this->paginator->count()`. We need to allow passing in a pre-computed total for cases where the Paginator's built-in count is inaccurate.

**Approach**: Add an optional `?int $totalOverride = null` parameter to `PaginatedResult`. When provided, use it instead of `$this->paginator->count()`.

### Decision 3: File locations

- `src/Repository/PaginatedResult.php` — add `$totalOverride` parameter
- `src/Repository/ClassesRepository.php` — modify `searchQueryPaginated()` to compute a separate count and pass it as `$totalOverride`

## Risks / Trade-offs

- **Risk**: The separate count query adds one additional SQL query per request. **Mitigation**: The count query uses `COUNT(DISTINCT cl.id)` which is simple and uses the same indexes as the main query. The performance impact is negligible.
- **Risk**: Filters need to be applied consistently to both the main query and the count query. **Mitigation**: The count query is cloned from the main query builder after filters are applied but before pagination, ensuring consistency.
- **Trade-off**: We modify `PaginatedResult` which is used by other paginated queries. **Mitigation**: The `$totalOverride` is optional and defaults to `null`, preserving existing behavior for all other callers.
