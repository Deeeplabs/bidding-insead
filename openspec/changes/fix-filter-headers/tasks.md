## 1. Fix GROUP BY + Paginator SQL Error in ClassesRepository (Courses Endpoint)

- [x] 1.1 In `bidding-api/src/Repository/ClassesRepository.php`, locate `searchQueryPaginated()` method. Find the `PaginatedResult` construction (line ~735). Pass a variable `$fetchJoinCollection` to the Paginator constructor instead of hardcoded `true`.
- [x] 1.2 Set `$fetchJoinCollection = !$campaignGroupMode` so that when `campaignGroupMode` is true, `fetchJoinCollection` is `false`. This prevents the Paginator's `LimitSubqueryOutputWalker` from wrapping the query in a subquery that includes non-grouped LEFT JOIN columns (`cp`, `p`, `pp`).
- [x] 1.3 Verify that the `$totalOverride` (already computed via `COUNT(DISTINCT cl.id)`) is still passed correctly, ensuring pagination metadata remains accurate.
- [ ] 1.4 Test `GET /v2/api/courses?page=1&limit=10&sort=credits&order=ASC` to confirm no SQL error 1055.
- [ ] 1.5 Test `GET /v2/api/courses?page=1&limit=10` (no sort) to confirm default ordering still works.

## 2. Fix Course Sort Field Mapping (`course_class_section` not allowed)

- [x] 2.1 In `bidding-api/src/Controller/Api/Course/CourseController.php`, locate `filterCourses()` method's `$sortFieldMap` (line ~315). Add `'course_class_section' => 'cl.section'` to the map so the frontend field name is translated to the correct DQL column.
- [x] 2.2 In `bidding-api/src/Domain/Course/CourseListQueryValidator.php`, update `sortConstraints()` to add `SortValidationConstraint::any('ct.name')` and `SortValidationConstraint::any('cl.section')` so both type and section sorting pass validation.
- [ ] 2.3 Test `GET /v2/api/courses?page=1&limit=10&sort=course_class_section&order=ASC` — must return HTTP 200, no "not allowed for sorting" error.
- [ ] 2.4 Test `GET /v2/api/courses?page=1&limit=10&sort=type&order=ASC` — must return HTTP 200, no validation error (was also broken because `ct.name` was missing from validator).

## 3. Add `class_id` to Course List Response

- [x] 3.1 In `bidding-api/src/Domain/Course/CourseDto.php`, add a new property `public ?int $class_id = null;` with an `#[OA\Property]` attribute for API documentation.
- [x] 3.2 In `bidding-api/src/Controller/Api/Course/CourseController.php`, in the `filterCourses()` method's class iteration loop (line ~347), add `$courseDto->class_id = $class->getId();` after the mapper call.
- [x] 3.3 In `bidding-admin/src/src/course/course-response.ts`, add `class_id: number;` to the `SingleCourse` type.
- [ ] 3.4 Test `GET /v2/api/courses?page=1&limit=10` — verify each item in the response now includes a `class_id` field with an integer value.

## 4. Fix DISTINCT + ORDER BY SQL Error in StudentRepository

- [x] 4.1 In `bidding-api/src/Repository/StudentRepository.php`, locate `queryPaginated()` method (line ~131). Replace `->select("DISTINCT $alias")` with `->select($alias)` and add `->groupBy("$alias.id")` to eliminate the DISTINCT keyword while still deduplicating rows.
- [x] 4.2 Verify that the Paginator (instantiated with `fetchJoinCollection: false` at line ~232) works correctly with the GROUP BY approach — count query should still return accurate totals.
- [ ] 4.3 Test sorting by `promotion_name` (maps to `p.label`), `programme_name` (maps to `prog.name`), and `home_campus` (maps to `c.label`) via the API to confirm no SQL 3065 error.

## 5. Fix Frontend Sort Field Mapping in Student Settings Table

- [x] 5.1 In `bidding-admin/src/components/settings/student-setting-table.tsx`, update the `fieldMapping` object to use correct backend field names:
  - `id` → `'people_soft_id'` (was `'id'`)
  - `name` → `'first_name'` (was `'full_name'`)
  - `program` → `'programme_name'` (was `'program'`)
- [x] 5.2 In `bidding-admin/src/app/(authenticated)/mba/settings/students/page.tsx`, fix `handleSort` (line ~298) to send sort as a simple string field name matching backend expectations (e.g., `first_name`, `promotion_name`) instead of DQL paths like `s.lastName`.
- [x] 5.3 Update the default `currentSort` state (line ~33) from `{ sort: 's.lastName', order: 'asc' }` to `{ sort: 'first_name', order: 'asc' }`.
- [x] 5.4 Fix `handleSearch` (line ~194) and `handleChangePage` (line ~280) and the initial `useEffect` (line ~323) to send `sort` as a simple string, not a `Sort` object with `{ field, direction }`.

## 6. Standardize Course Table Headers

- [x] 6.1 In `bidding-admin/src/components/dashboard/process/bidding-round/course-table-bidding-round.tsx`, change column label `'Seat Available'` → `'Seats Available'` (line ~48) and `'Course'` → `'Course Name'` in the render column (line ~36 label).
- [x] 6.2 In `bidding-admin/src/components/dashboard/process/add-drop/course-table.tsx`, change column label `'Credit'` → `'Credits'` (line ~47) and `'Course'` → `'Course Name'` in the render column (line ~35 label).
- [x] 6.3 In `bidding-admin/src/components/settings/course-table-setting.tsx`, change column label `'Course'` → `'Course Name'` (line ~150).

## 8. Add In-Memory Sorting for Computed Columns in CourseController (Backend)

- [x] 8.1 In `bidding-api/src/Controller/Api/Course/CourseController.php`, in `filterCourses()`, define a `$computedSortFields` array: `['seat', 'conflict', 'fallback', 'promotions']`. Add a boolean `$isComputedSort = $request->sort !== null && in_array($request->sort, $computedSortFields, true);`.
- [x] 8.2 When `$isComputedSort` is true, override pagination to fetch all records: set `$limit` to `$maxNoPaginationLimit` (10000) and `$offset` to `0`. Do NOT pass `$sort` to `classService->getList()` (set `$sort = null` for the query, since sorting will happen in PHP).
- [x] 8.3 After the existing `foreach` loop that builds `$response->items` (after line ~405), add the in-memory sort block:
  - Use `usort($response->items, ...)` with a comparator that reads `$courseDto->{$request->sort}` for each item.
  - Handle `null` values by pushing them to the end of the sorted list.
  - For `promotions` (string field), use `strcasecmp()`. For numeric fields (`seat`, `conflict`, `fallback`), use spaceship operator (`<=>`).
  - Respect `$request->order` (ASC/DESC).
- [x] 8.4 After sorting, slice the array for the requested page:
  - `$requestedPage = $request->page > 0 ? $request->page : 1;`
  - `$requestedLimit = $request->limit > 0 ? $request->limit : 20;`
  - `$requestedOffset = ($requestedPage - 1) * $requestedLimit;`
  - `$totalRecords = count($response->items);`
  - `$response->items = array_slice($response->items, $requestedOffset, $requestedLimit);`
  - Rebuild `$response->pagination` with correct `totalRecords`, `currentPage`, and `totalPages`.
- [x] 8.5 Ensure the non-computed sort path (existing SQL-level sort) is unaffected — wrap the computed sort logic in the `if ($isComputedSort)` block only.
- [ ] 8.6 Test `GET /v2/api/courses?page=1&limit=10&sort=seat&order=ASC` — must return HTTP 200 with courses sorted by available seats ascending.
- [ ] 8.7 Test `GET /v2/api/courses?page=1&limit=10&sort=conflict&order=DESC` — must return HTTP 200 with courses sorted by conflict count descending.
- [ ] 8.8 Test `GET /v2/api/courses?page=2&limit=10&sort=seat&order=ASC` — verify page 2 returns items 10–19 from the sorted set with correct pagination metadata.

## 9. Make seat/conflict/fallback/promotions Sortable in Frontend Course Table

- [x] 9.1 In `bidding-admin/src/components/settings/course-table-setting.tsx`, update `SortField` type to include the 4 new fields: `type SortField = 'name' | 'credits' | 'type' | 'seat' | 'conflict' | 'fallback' | 'promotions';`
- [x] 9.2 Add the 4 new fields to `fieldMapping`: `seat: 'seat'`, `conflict: 'conflict'`, `fallback: 'fallback'`, `promotions: 'promotions'`.
- [x] 9.3 Add the 4 new fields to `backendSortableFields` array.
- [x] 9.4 Update the Seats column `<th>` to be clickable with sort handler: add `cursor-pointer hover:bg-gray-200` classes, `onClick={() => handleSort('seat')}`, and sort icon via `getSortIcon('seat')`.
- [x] 9.5 Update the Conflicts column `<th>` similarly with `onClick={() => handleSort('conflict')}` and sort icon.
- [x] 9.6 Update the Fallbacks column `<th>` similarly with `onClick={() => handleSort('fallback')}` and sort icon.
- [x] 9.7 Update the Promotions column `<th>` similarly with `onClick={() => handleSort('promotions')}` and sort icon.
- [x] 9.8 Add sort cases in the `sortedCourses` `useMemo` switch statement for `seat`, `conflict`, `fallback` (numeric comparison) and `promotions` (string comparison, handle null).
- [ ] 9.9 In the admin UI, navigate to Settings > Courses, click each of the 4 new sortable column headers and verify the sort icon toggles and data re-sorts correctly.

## 10. Verification

- [ ] 10.1 Test `GET /v2/api/courses?page=1&limit=10&sort=credits&order=ASC` — must return HTTP 200 with sorted courses, no SQL error 1055.
- [ ] 10.2 Test `GET /v2/api/courses?page=1&limit=10&sort=name&order=ASC` — must return HTTP 200.
- [ ] 10.3 Test `GET /v2/api/courses?page=1&limit=10&sort=type&order=DESC` — must return HTTP 200.
- [ ] 10.4 Test `GET /v2/api/courses?page=1&limit=10&sort=course_class_section&order=ASC` — must return HTTP 200 with courses sorted by section.
- [ ] 10.5 Test `GET /v2/api/courses?page=1&limit=10` — verify each item includes `class_id` field with an integer value.
- [ ] 10.6 Test the student list API endpoint with sort params `sort=promotion_name&order=asc`, `sort=programme_name&order=asc`, `sort=home_campus&order=asc` to confirm HTTP 200 with sorted results.
- [ ] 10.7 In the admin UI, navigate to Settings > Students and click each sortable column header to verify sorting works without "No records found!" errors.
- [ ] 10.8 Verify that sorting combined with filters (e.g., filter by promotion + sort by home campus) returns correct filtered and sorted results.
- [ ] 10.9 Verify course table headers display correctly in Bidding Round and Add-Drop dashboard views.
- [ ] 10.10 Test `GET /v2/api/courses?page=1&limit=10&sort=seat&order=ASC` — verify courses are sorted by available seats ascending.
- [ ] 10.11 Test `GET /v2/api/courses?page=1&limit=10&sort=seat&order=DESC` — verify courses are sorted by available seats descending.
- [ ] 10.12 Test `GET /v2/api/courses?page=1&limit=10&sort=conflict&order=DESC` — verify courses are sorted by conflict count descending.
- [ ] 10.13 Test `GET /v2/api/courses?page=1&limit=10&sort=promotions&order=ASC` — verify courses are sorted alphabetically by promotion labels.
- [ ] 10.14 Test `GET /v2/api/courses?page=2&limit=10&sort=seat&order=ASC` — verify page 2 pagination is correct for computed sort.
- [ ] 10.15 Test `GET /v2/api/courses?page=1&limit=10&sort=seat&order=DESC&min_credits=1` — verify computed sort works with filters.
- [ ] 10.16 In the admin UI, navigate to Settings > Courses and click Seats, Conflicts, Fallbacks, Promotions column headers — verify sort icons toggle and data re-sorts.
- [ ] 10.17 Smoke test: navigate across all dashboard views that show student and course tables to confirm no regressions.
