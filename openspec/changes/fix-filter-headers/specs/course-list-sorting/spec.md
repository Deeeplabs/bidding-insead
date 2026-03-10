## course-list-sorting

Fix sorting on the courses list endpoint so that all sortable columns work without triggering SQL error 1055 (only_full_group_by).

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

### API Endpoint

- **Method**: GET/POST `/v2/api/courses` (mapped to `CourseController::filterCourses()`)
- **Sort parameters**: `sort` (field name: name, credits, credit, type, id, course_class_section) + `order` (ASC/DESC)
- **Root cause 1 (SQL 1055)**: `ClassesRepository::searchQueryPaginated()` line ~697 uses `GROUP BY cl.id, c.id, s.id, ct.id, cam.id, stat.id` in campaign mode, but Doctrine Paginator with `fetchJoinCollection: true` wraps the query in a subquery that includes columns from LEFT JOINed `cp`, `p`, `pp` tables not in GROUP BY
- **Fix location 1**: `ClassesRepository::searchQueryPaginated()` — set `fetchJoinCollection` to `false` when `campaignGroupMode` is true
- **Root cause 2 (sort mapping)**: `CourseController::filterCourses()` `$sortFieldMap` missing `course_class_section` entry; `CourseListQueryValidator::sortConstraints()` missing `ct.name` and `cl.section`
- **Fix location 2**: `CourseController::filterCourses()` `$sortFieldMap` + `CourseListQueryValidator::sortConstraints()`
- **Response shape**: `class_id` (integer) added to each course list item — additive, non-breaking
