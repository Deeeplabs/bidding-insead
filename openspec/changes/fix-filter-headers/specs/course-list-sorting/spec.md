## course-list-sorting

Fix sorting on the courses list endpoint so that all sortable columns work without triggering SQL error 1055 (only_full_group_by). Add sorting support for computed columns (seat, conflict, fallback, promotions) via in-memory sorting.

### Scenario: Sort courses by credits ascending

- **Given** the admin calls `GET /v2/api/courses?page=1&limit=10&sort=credits&order=ASC`
- **When** the backend processes the request through `ClassesRepository::searchQueryPaginated()` in campaign group mode
- **Then** the API returns courses sorted by credit value in ascending order
- **And** no SQL error 1055 occurs
- **And** the response contains correct pagination metadata

### Scenario: Sort courses by credits descending

- **Given** the admin calls `GET /v2/api/courses?page=1&limit=10&sort=credits&order=DESC`
- **When** the backend processes the request
- **Then** the API returns courses sorted by credit value in descending order
- **And** no SQL error 1055 occurs

### Scenario: Sort courses by name

- **Given** the admin calls `GET /v2/api/courses?page=1&limit=10&sort=name&order=ASC`
- **When** the backend processes the request
- **Then** the API returns courses sorted by course name in ascending order
- **And** no SQL error occurs

### Scenario: Sort courses by type

- **Given** the admin calls `GET /v2/api/courses?page=1&limit=10&sort=type&order=ASC`
- **When** the backend processes the request and maps `type` → `ct.name` via `$sortFieldMap`, then validates against `CourseListQueryValidator::sortConstraints()` which includes `ct.name`
- **Then** the API returns courses sorted by course type name in ascending order
- **And** no SQL error or validation error occurs

### Scenario: Sort courses by section (course_class_section)

- **Given** the admin calls `GET /v2/api/courses?page=1&limit=10&sort=course_class_section&order=ASC`
- **When** the backend processes the request and maps `course_class_section` → `cl.section` via `$sortFieldMap`, then validates against `CourseListQueryValidator::sortConstraints()` which includes `cl.section`
- **Then** the API returns courses sorted by class section in ascending order
- **And** no "Field not allowed for sorting" error occurs

### Scenario: Course list response includes class_id

- **Given** the admin calls `GET /v2/api/courses?page=1&limit=10`
- **When** the backend processes the request through `CourseController::filterCourses()` and iterates over Class entities
- **Then** each item in the response includes a `class_id` field with the integer ID of the Class entity
- **And** the `class_id` is distinct per class section row

### Scenario: Sort courses with filters applied

- **Given** the admin calls the courses endpoint with both sort and filter parameters (e.g., `sort=credits&order=ASC&min_credits=1&max_credits=3`)
- **When** the backend processes the request
- **Then** the API returns filtered courses sorted by the specified field
- **And** no SQL error occurs
- **And** the pagination count reflects only filtered results

### Scenario: Default pagination still works without sort

- **Given** the admin calls `GET /v2/api/courses?page=1&limit=10` without sort parameters
- **When** the backend processes the request
- **Then** the API returns courses with default ordering (by name ASC)
- **And** no SQL error occurs

### Scenario: Sort courses by seat (computed field) ascending

- **Given** the admin calls `GET /v2/api/courses?page=1&limit=10&sort=seat&order=ASC`
- **When** the backend detects `seat` is a computed sort field
- **Then** the backend fetches all matching classes without SQL-level pagination or sorting
- **And** computes `seat` for each class (totalSeats - enrolled - invited - waitlisted)
- **And** sorts the full result set by `seat` ascending in PHP
- **And** slices the sorted array for page 1 with limit 10
- **And** returns correct pagination metadata reflecting total record count

### Scenario: Sort courses by seat descending

- **Given** the admin calls `GET /v2/api/courses?page=1&limit=10&sort=seat&order=DESC`
- **When** the backend processes the request with in-memory sorting
- **Then** the API returns courses sorted by available seats in descending order (highest seats first)
- **And** pagination metadata is correct

### Scenario: Sort courses by conflict (computed field)

- **Given** the admin calls `GET /v2/api/courses?page=1&limit=10&sort=conflict&order=DESC`
- **When** the backend detects `conflict` is a computed sort field
- **Then** the backend fetches all matching classes, computes conflict count for each (from comma-separated `classConflicts` string)
- **And** sorts by conflict count descending in PHP
- **And** returns the correct page slice with accurate pagination

### Scenario: Sort courses by fallback (computed field)

- **Given** the admin calls `GET /v2/api/courses?page=1&limit=10&sort=fallback&order=ASC`
- **When** the backend detects `fallback` is a computed sort field
- **Then** the backend fetches all records, computes fallback (currently always 0), sorts in PHP
- **And** returns results (effectively unsorted since all values are 0) with correct pagination

### Scenario: Sort courses by promotions (computed string field)

- **Given** the admin calls `GET /v2/api/courses?page=1&limit=10&sort=promotions&order=ASC`
- **When** the backend detects `promotions` is a computed sort field
- **Then** the backend fetches all records, computes promotion labels for each class
- **And** sorts alphabetically by the comma-separated promotion string in PHP
- **And** null/empty promotions are pushed to the end of the result set
- **And** returns the correct page slice with accurate pagination

### Scenario: Computed sort with filters applied

- **Given** the admin calls `GET /v2/api/courses?page=1&limit=10&sort=seat&order=DESC&min_credits=1&campus_id[]=2`
- **When** the backend processes filters at SQL level and sorting at PHP level
- **Then** the SQL query applies credit and campus filters to reduce the result set
- **And** the in-memory sort applies only to the filtered results
- **And** pagination metadata reflects the filtered total, not the full table count

### Scenario: Computed sort with page 2

- **Given** the admin calls `GET /v2/api/courses?page=2&limit=10&sort=seat&order=ASC`
- **When** the backend fetches all records, computes seat for each, sorts in PHP
- **Then** `array_slice` returns items 10–19 from the sorted array
- **And** pagination shows `currentPage: 2` with correct `totalPages`

### Scenario: Frontend seat column header is sortable

- **Given** the admin views the course list table in Settings
- **When** the admin clicks the "Seats" column header
- **Then** the sort icon changes to ascending (↑)
- **And** the frontend sends `sort=seat&order=ASC` to the backend
- **And** clicking again toggles to descending (↓), then clears sort (↕)

### Scenario: Frontend conflict column header is sortable

- **Given** the admin views the course list table in Settings
- **When** the admin clicks the "Conflicts" column header
- **Then** the frontend sends `sort=conflict&order=ASC` to the backend
- **And** the table re-renders with courses sorted by conflict count

### Scenario: Frontend fallback column header is sortable

- **Given** the admin views the course list table in Settings
- **When** the admin clicks the "Fallbacks" column header
- **Then** the frontend sends `sort=fallback&order=ASC` to the backend

### Scenario: Frontend promotions column header is sortable

- **Given** the admin views the course list table in Settings
- **When** the admin clicks the "Promotions" column header
- **Then** the frontend sends `sort=promotions&order=ASC` to the backend
- **And** the table re-renders with courses sorted alphabetically by promotion labels

### Scenario: Sort classes by promotions via ClassController (campaign class list)

- **Given** the admin views the campaign class list which calls `GET /v2/api/classes?sort=promotions&order=ASC&campaign_id=1&promotion_id=1`
- **When** the backend detects `promotions` is a computed sort field in `ClassController::filterClasses()`
- **Then** the backend fetches all matching classes, computes promotion labels for each
- **And** sorts alphabetically by the comma-separated promotion string in PHP
- **And** returns the correct page slice with accurate pagination
- **And** no `Field "promotions" is not allowed for sorting` error occurs

### API Endpoints

**Course endpoint** (`CourseController::filterCourses()`):
- **Method**: GET/POST `/v2/api/courses`
- **Sort parameters**: `sort` (field name: name, credits, credit, type, id, course_class_section, seat, conflict, fallback, promotions) + `order` (ASC/DESC)
- **SQL-sortable fields**: name, credits, credit, type, id, course_class_section — sorted via DQL ORDER BY
- **Computed-sortable fields**: seat, conflict, fallback, promotions — sorted via PHP in-memory sort after fetching all records

**Class endpoint** (`ClassController::filterClasses()`):
- **Method**: GET/POST `/v2/api/classes`
- **Sort parameters**: `sort` (field name: section, id, name, course, campus_course_name, course_id, credit, credits, course_credit, campus, campus_name, deadline, total_seats, total_conflicts, total_fallback, promotions) + `order` (ASC/DESC)
- **SQL-sortable fields**: section, id, name, course, campus_course_name, course_id, credit, credits, course_credit, campus, campus_name, deadline — mapped via `$fieldMapping` then sorted via DQL ORDER BY
- **Computed-sortable fields**: total_seats, total_conflicts, total_fallback, promotions — sorted via PHP in-memory sort after fetching all records

**Root causes and fixes**:
- **Root cause 1 (SQL 1055)**: `ClassesRepository::searchQueryPaginated()` line ~697 uses `GROUP BY cl.id, c.id, s.id, ct.id, cam.id, stat.id` in campaign mode, but Doctrine Paginator with `fetchJoinCollection: true` wraps the query in a subquery that includes columns from LEFT JOINed `cp`, `p`, `pp` tables not in GROUP BY
- **Fix location 1**: `ClassesRepository::searchQueryPaginated()` — set `fetchJoinCollection` to `false` when `campaignGroupMode` is true
- **Root cause 2 (sort mapping)**: `CourseController::filterCourses()` `$sortFieldMap` missing `course_class_section` entry; `CourseListQueryValidator::sortConstraints()` missing `ct.name` and `cl.section`
- **Fix location 2**: `CourseController::filterCourses()` `$sortFieldMap` + `CourseListQueryValidator::sortConstraints()`
- **Fix location 3**: `CourseController::filterCourses()` — add computed-field detection and in-memory sorting with post-sort pagination
- **Fix location 4**: `course-table-setting.tsx` — add seat, conflict, fallback, promotions to `SortField`, `fieldMapping`, `backendSortableFields`, and column header click handlers
- **Fix location 5**: `ClassController::filterClasses()` — add `promotions` to `$computedSortFields` array and add `promotions` case to computed sort switch statement
- **Response shape**: `class_id` (integer) added to each course list item — additive, non-breaking
