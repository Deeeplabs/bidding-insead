## Why

When sorting student or course table columns in the admin dashboard, certain sort fields trigger a MySQL 8 error (SQLSTATE 3065): "Expression #1 of ORDER BY clause is not in SELECT list, references column 'bidding.p2_.label' which is not in SELECT list; this is incompatible with DISTINCT". This causes a 500 Internal Server Error, which the frontend interprets as an empty result set, displaying "No records found!" — making it appear that filters/sorting broke the data.

**Root cause**: `StudentRepository::queryPaginated()` uses `SELECT DISTINCT s` (Doctrine entity). When the `FilterableQueryProvider` applies an `ORDER BY` on a joined entity's column (e.g., `p.label` for Promotion, `c.label` for Campus, `prog.name` for Programme), MySQL 8's strict mode rejects it because the ORDER BY column isn't in the SELECT DISTINCT list.

Additionally, course and student table headers across different dashboard views use inconsistent naming (e.g., "Credit" vs "Credits", "Seat Available" vs "Seats Available", "Course" vs "Course Name").

## What Changes

- **Fix DISTINCT + ORDER BY conflict**: Modify `StudentRepository::queryPaginated()` to add joined entity sort columns to the SELECT clause, or replace DISTINCT with GROUP BY, so that MySQL 8 allows ORDER BY on joined columns.
- **Fix frontend sort field mapping**: Ensure the `student-setting-table.tsx` `fieldMapping` values match what `StudentService::listStudents()` expects (e.g., `full_name` → should be `first_name`, `id` → should be `people_soft_id`, `program` → should be `programme_name`).
- **Standardize table headers**: Normalize column labels across course and student tables in `bidding-admin` for consistency (e.g., always "Credits" not "Credit", always "Seats Available" not "Seat Available").

## Capabilities

### Modified Capabilities
- `student-list-sorting`: Fix backend query to support sorting by Promotion, Programme, and Home Campus columns without triggering SQL error 3065.
- `student-list-headers`: Standardize student table column headers across all dashboard views.
- `course-list-headers`: Standardize course table column headers across all dashboard views (Pre-Bidding, Bidding Round, Add-Drop, Settings).

## Impact

- **`bidding-api/src/Repository/StudentRepository.php`**: `queryPaginated()` method — must fix DISTINCT + ORDER BY incompatibility for joined entity columns (`p.label`, `c.label`, `prog.name`).
- **`bidding-admin/src/components/settings/student-setting-table.tsx`**: Fix `fieldMapping` to send correct backend sort field names. Fix inconsistent frontend-backend sort field mapping.
- **`bidding-admin/src/app/(authenticated)/mba/settings/students/page.tsx`**: Fix `handleSort` to send sort as a simple string field matching what `StudentController` expects, not a `Sort` object with DQL paths like `s.lastName`.
- **`bidding-admin/src/components/dashboard/process/add-drop/course-table.tsx`**: Standardize column labels.
- **`bidding-admin/src/components/dashboard/process/bidding-round/course-table-bidding-round.tsx`**: Standardize column labels.
- **`bidding-admin/src/components/settings/course-table-setting.tsx`**: Standardize column labels if needed.
- No database schema changes, no migration required.
- No API response shape changes — purely backend query fix and frontend label/mapping corrections.
