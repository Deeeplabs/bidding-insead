# Fix SQL Errors on Student & Course List Sorting + Sort Field Mapping + Add `class_id` + Computed Column Sorting

## Problems

### 1. Course list sorting — SQL Error 1055 (only_full_group_by)

`GET /v2/api/courses?page=1&limit=10&sort=credits&order=ASC` returns a 500 error:

> SQLSTATE[42000]: Syntax error or access violation: 1055 Expression #19 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'bidding.c2_.id' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by

**Root cause**: `ClassesRepository::searchQueryPaginated()` in campaign group mode uses `GROUP BY cl.id, c.id, s.id, ct.id, cam.id, stat.id` but has LEFT JOINs to `classPromotions (cp)`, `promotion (p)`, `promotionPeriods (pp)`. The Doctrine Paginator with `fetchJoinCollection: true` internally wraps the query in a subquery whose SELECT includes columns from those LEFT JOINed tables — columns not covered by the GROUP BY — causing the MySQL violation.

### 2. Student list sorting — SQL Error 3065 (DISTINCT + ORDER BY)

Sorting the student list by joined entity columns (Promotion, Programme, Home Campus) returns a 500 error with MySQL 8 error 3065: "Expression #1 of ORDER BY clause is not in SELECT list … incompatible with DISTINCT".

**Root cause**: `StudentRepository::queryPaginated()` uses `SELECT DISTINCT s`. When `FilterableQueryProvider` applies `ORDER BY` on a joined column (e.g., `p.label`), MySQL rejects it because the ORDER BY column is not in the SELECT DISTINCT list.

### 3. Course list sort field mapping — `course_class_section` not allowed

Sorting the course list by section triggers: `Field "course_class_section" is not allowed for sorting`.

**Root cause**: `CourseController::filterCourses()` `$sortFieldMap` did not include `course_class_section`, so the value passed through unmapped. `CourseListQueryValidator::sortConstraints()` only allowed `c.name`, `c.id`, `c.credit` — missing `cl.section` (for section) and `ct.name` (for type). Type sorting was also broken for the same reason.

### 4. Missing `class_id` in course list response

The course list endpoint iterates over Class entities but does not expose the class ID in the response, making it impossible for the frontend to uniquely identify class sections.

### 5. Course list — No sorting support for computed columns

The course list table columns `seat`, `conflict`, `fallback`, and `promotions` are computed in PHP after the SQL query (seat = totalSeats - enrolled - invited - waitlisted; conflict = count of conflict IDs; fallback = hardcoded 0; promotions = comma-separated labels). These fields cannot be sorted at the SQL/DQL level, so clicking these column headers in the admin UI had no backend sort effect.

## Solution

### 1. Course list — Disable `fetchJoinCollection` in campaign group mode

Set `fetchJoinCollection: !$campaignGroupMode` on the Doctrine Paginator. When campaign group mode is active, `GROUP BY cl.id` already ensures one row per class, making the Paginator's subquery-wrapping (which causes the column conflict) unnecessary. The total count is already computed separately via `COUNT(DISTINCT cl.id)` and passed as `totalOverride`.

### 2. Student list — Replace `DISTINCT` with `GROUP BY`

Replaced `SELECT DISTINCT` with `GROUP BY` on the primary key, which achieves the same row deduplication without imposing the SELECT-list restriction on ORDER BY columns.

### 3. Course list — Fix sort field mapping

Added `'course_class_section' => 'cl.section'` to `$sortFieldMap` in `CourseController::filterCourses()`. Added `ct.name` and `cl.section` to `CourseListQueryValidator::sortConstraints()` so both type and section sorting pass validation.

### 4. Course list — Add `class_id` to response

Added `class_id` property to `CourseDto`, populated from `$class->getId()` in the `filterCourses()` iteration loop.

### 5. Course list — In-memory sorting for computed columns (seat, conflict, fallback, promotions)

When sorting by a computed field (`seat`, `conflict`, `fallback`, `promotions`), the controller:
1. Detects the sort field is computed (not a SQL column)
2. Fetches all matching records (overrides pagination to fetch everything)
3. Builds all DTOs with computed values as usual
4. Sorts the full array in PHP using `usort()` with appropriate comparators (numeric spaceship `<=>` for seat/conflict/fallback, `strcasecmp()` for promotions)
5. Slices the sorted array for the requested page via `array_slice()`
6. Rebuilds pagination metadata from the total count

Null values are pushed to the end of the sorted list regardless of sort direction.

## Changes Made

1. **`src/Repository/ClassesRepository.php`**
   - **Method**: `searchQueryPaginated()`
   - Changed Paginator construction from `fetchJoinCollection: true` to `fetchJoinCollection: !$campaignGroupMode`
   - When `campaignGroupMode` is true (set by `/v2/api/courses` endpoint), the Paginator no longer wraps the query in a subquery, avoiding the GROUP BY / only_full_group_by conflict

2. **`src/Repository/StudentRepository.php`**
   - **Method**: `queryPaginated()`
   - Replaced `->select("DISTINCT $alias")` with `->select($alias)` + `->groupBy("$alias.id")`
   - `GROUP BY s.id` deduplicates rows while allowing ORDER BY on any joined column
   - Paginator already uses `fetchJoinCollection: false`, so count query works correctly

3. **`src/Controller/Api/Course/CourseController.php`**
   - **Method**: `filterCourses()`
   - Added `'course_class_section' => 'cl.section'` to `$sortFieldMap` so the frontend field name is translated to the correct DQL column
   - Added `$courseDto->class_id = $class->getId();` in the class iteration loop to populate the new field

4. **`src/Domain/Course/CourseListQueryValidator.php`**
   - **Method**: `sortConstraints()`
   - Added `SortValidationConstraint::any('ct.name')` — allows sorting by course type name
   - Added `SortValidationConstraint::any('cl.section')` — allows sorting by class section

5. **`src/Domain/Course/CourseDto.php`**
   - Added `public ?int $class_id = null;` property with `#[OA\Property]` attribute

6. **`src/Controller/Api/Course/CourseController.php`** (additional changes for computed sorting)
   - **Method**: `filterCourses()`
   - Added `$computedSortFields` array and `$isComputedSort` detection
   - When sorting by computed field: overrides `$limit` to 10000 and `$offset` to 0 to fetch all records; skips SQL-level `Sort` (sets `$sort = null`)
   - After the DTO-building loop: `usort()` sorts `$response->items` by the computed field with direction support
   - Post-sort: `array_slice()` for pagination, rebuilds `$response->pagination` with correct totals

## Impact

- **API response shape change**: `class_id` (integer) field added to course list items — additive, non-breaking
- **Sort behavior change**: Sorting by `seat`, `conflict`, `fallback`, `promotions` uses in-memory sorting (all filtered records fetched, sorted in PHP, then paginated). Non-computed sorts (name, credits, type, section) continue to use SQL-level ORDER BY.
- **No migration required**: No database schema changes
- **Backward compatible**: SQL fixes are functionally equivalent to previous behavior — only the SQL generation strategy changes
- **Scoped**: The `fetchJoinCollection` change only affects the campaign group mode path; non-campaign queries retain `fetchJoinCollection: true`

## Verification

- [ ] `GET /v2/api/courses?page=1&limit=10&sort=credits&order=ASC` — returns 200 with sorted courses (no SQL 1055)
- [ ] `GET /v2/api/courses?page=1&limit=10&sort=name&order=ASC` — returns 200
- [ ] `GET /v2/api/courses?page=1&limit=10&sort=type&order=DESC` — returns 200 (was also broken, now fixed)
- [ ] `GET /v2/api/courses?page=1&limit=10&sort=course_class_section&order=ASC` — returns 200 with courses sorted by section
- [ ] `GET /v2/api/courses?page=1&limit=10` — returns 200, each item includes `class_id` field
- [ ] `GET /v2/api/courses?page=1&limit=10&sort=seat&order=ASC` — returns 200, sorted by available seats ascending
- [ ] `GET /v2/api/courses?page=1&limit=10&sort=seat&order=DESC` — returns 200, sorted by available seats descending
- [ ] `GET /v2/api/courses?page=1&limit=10&sort=conflict&order=DESC` — returns 200, sorted by conflict count descending
- [ ] `GET /v2/api/courses?page=1&limit=10&sort=promotions&order=ASC` — returns 200, sorted alphabetically by promotion labels
- [ ] `GET /v2/api/courses?page=2&limit=10&sort=seat&order=ASC` — returns 200, page 2 pagination correct for computed sort
- [ ] `GET /v2/api/courses?page=1&limit=10&sort=seat&order=DESC&min_credits=1` — computed sort works with filters
- [ ] Sort student list by Promotion (`sort=promotion_name&order=asc`) — returns 200
- [ ] Sort student list by Programme (`sort=programme_name&order=asc`) — returns 200
- [ ] Sort student list by Home Campus (`sort=home_campus&order=asc`) — returns 200
- [ ] Pagination totals are correct for both endpoints
- [ ] Sorting combined with filters returns correct results
