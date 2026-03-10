## Why

Multiple SQL errors occur when sorting admin dashboard tables:

1. **Student list sorting (SQL 3065)**: Sorting student table columns by joined entity fields (Promotion, Programme, Campus) triggers MySQL 8 error SQLSTATE 3065 — "Expression #1 of ORDER BY clause is not in SELECT list … incompatible with DISTINCT". Root cause: `StudentRepository::queryPaginated()` uses `SELECT DISTINCT s`; when `FilterableQueryProvider` applies `ORDER BY` on a joined column (e.g., `p.label`), MySQL rejects it.

2. **Course list sorting (SQL 1055)**: The courses endpoint `GET /v2/api/courses?sort=credits&order=ASC` triggers MySQL error SQLSTATE 42000 / 1055 — "Expression #19 of SELECT list is not in GROUP BY clause and contains nonaggregated column … incompatible with sql_mode=only_full_group_by". Root cause: `ClassesRepository::searchQueryPaginated()` uses `GROUP BY cl.id, c.id, s.id, ct.id, cam.id, stat.id` in campaign group mode, but the Doctrine Paginator with `fetchJoinCollection: true` internally generates SQL that includes columns from LEFT JOINed entities (`classPromotions`, `promotion`, `promotionPeriods`) in the SELECT list, which are not covered by the GROUP BY.

3. **Frontend inconsistencies**: Course and student table headers use inconsistent naming (e.g., "Credit" vs "Credits", "Seat Available" vs "Seats Available"), and student settings sort field mapping sends incorrect field names to the backend.

4. **Course list sort field mapping incomplete**: Sorting the course list by `course_class_section` (Section column) triggers `Field "course_class_section" is not allowed for sorting`. Root cause: `CourseController::filterCourses()` `$sortFieldMap` does not include `course_class_section`, so the value passes through unmapped. Additionally, `CourseListQueryValidator::sortConstraints()` only allows `c.name`, `c.id`, `c.credit` — it is missing `cl.section` (for section) and `ct.name` (for type).

5. **Missing `class_id` in course list response**: The course list endpoint returns `CourseDto` objects that lack a `class_id` field. Since `filterCourses()` iterates over Class entities (not Course entities), each response item represents a class section, but the class ID is not exposed in the response.

## What Changes

- **Fix course list GROUP BY conflict**: In `ClassesRepository::searchQueryPaginated()`, when `campaignGroupMode` is true, set the Doctrine Paginator's `fetchJoinCollection` to `false` (since `GROUP BY cl.id` already deduplicates rows), preventing the Paginator from generating a wrapping subquery that includes non-grouped columns.
- **Fix student list DISTINCT + ORDER BY conflict**: Modify `StudentRepository::queryPaginated()` to replace `SELECT DISTINCT` with `GROUP BY s.id`, so that MySQL allows ORDER BY on joined columns.
- **Fix frontend sort field mapping**: Ensure `student-setting-table.tsx` `fieldMapping` values match what `StudentService::listStudents()` expects.
- **Standardize table headers**: Normalize column labels across course and student tables in `bidding-admin`.
- **Fix course sort field mapping**: Add `course_class_section` → `cl.section` to the `$sortFieldMap` in `CourseController::filterCourses()`, and add `cl.section` and `ct.name` to `CourseListQueryValidator::sortConstraints()`.
- **Add `class_id` to course list response**: Add a `class_id` property to `CourseDto`, populate it in `CourseController::filterCourses()`, and add it to the frontend `SingleCourse` type.

## Capabilities

### Modified Capabilities
- `course-list-sorting`: Fix backend query in `ClassesRepository::searchQueryPaginated()` to support sorting courses by any field without triggering SQL error 1055 when in campaign group mode. Add missing sort field mappings (`course_class_section` → `cl.section`) and update `CourseListQueryValidator` to allow all mapped sort fields. Add `class_id` to the course list response DTO.
- `student-list-sorting`: Fix backend query to support sorting by Promotion, Programme, and Home Campus columns without triggering SQL error 3065.
- `student-list-headers`: Standardize student table column headers across all dashboard views.
- `course-list-headers`: Standardize course table column headers across all dashboard views (Pre-Bidding, Bidding Round, Add-Drop, Settings).

## Impact

- **`bidding-api/src/Repository/ClassesRepository.php`**: `searchQueryPaginated()` method — must fix Paginator `fetchJoinCollection` when `campaignGroupMode` is true to prevent GROUP BY / only_full_group_by conflict. The Paginator wrapping subquery includes columns from LEFT JOINed `cp`, `p`, `pp` tables that aren't in the campaign GROUP BY clause.
- **`bidding-api/src/Repository/StudentRepository.php`**: `queryPaginated()` method — must fix DISTINCT + ORDER BY incompatibility for joined entity columns (`p.label`, `c.label`, `prog.name`).
- **`bidding-admin/src/components/settings/student-setting-table.tsx`**: Fix `fieldMapping` to send correct backend sort field names.
- **`bidding-admin/src/app/(authenticated)/mba/settings/students/page.tsx`**: Fix `handleSort` to send sort as a simple string field matching what `StudentController` expects.
- **`bidding-admin/src/components/dashboard/process/add-drop/course-table.tsx`**: Standardize column labels.
- **`bidding-admin/src/components/dashboard/process/bidding-round/course-table-bidding-round.tsx`**: Standardize column labels.
- **`bidding-admin/src/components/settings/course-table-setting.tsx`**: Standardize column labels if needed.
- **`bidding-api/src/Domain/Course/CourseListQueryValidator.php`**: `sortConstraints()` — must add `cl.section` and `ct.name` to allowed sort fields.
- **`bidding-api/src/Domain/Course/CourseDto.php`**: Add `class_id` property.
- **`bidding-admin/src/src/course/course-response.ts`**: Add `class_id` to `SingleCourse` type.
- No database schema changes, no migration required.
- API response shape change: `class_id` field added to course list items (additive, non-breaking).
