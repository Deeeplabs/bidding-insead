## 1. Fix DISTINCT + ORDER BY SQL Error in StudentRepository

- [x] 1.1 In `bidding-api/src/Repository/StudentRepository.php`, locate `queryPaginated()` method (line ~131). Replace `->select("DISTINCT $alias")` with `->select($alias)` and add `->groupBy("$alias.id")` to eliminate the DISTINCT keyword while still deduplicating rows.
- [x] 1.2 Verify that the Paginator (instantiated with `fetchJoinCollection: false` at line ~232) works correctly with the GROUP BY approach — count query should still return accurate totals.
- [ ] 1.3 Test sorting by `promotion_name` (maps to `p.label`), `programme_name` (maps to `prog.name`), and `home_campus` (maps to `c.label`) via the API to confirm no SQL 3065 error.

## 2. Fix Frontend Sort Field Mapping in Student Settings Table

- [x] 2.1 In `bidding-admin/src/components/settings/student-setting-table.tsx`, update the `fieldMapping` object to use correct backend field names:
  - `id` → `'people_soft_id'` (was `'id'`)
  - `name` → `'first_name'` (was `'full_name'`)
  - `program` → `'programme_name'` (was `'program'`)
- [x] 2.2 In `bidding-admin/src/app/(authenticated)/mba/settings/students/page.tsx`, fix `handleSort` (line ~298) to send sort as a simple string field name matching backend expectations (e.g., `first_name`, `promotion_name`) instead of DQL paths like `s.lastName`.
- [x] 2.3 Update the default `currentSort` state (line ~33) from `{ sort: 's.lastName', order: 'asc' }` to `{ sort: 'first_name', order: 'asc' }`.
- [x] 2.4 Fix `handleSearch` (line ~194) and `handleChangePage` (line ~280) and the initial `useEffect` (line ~323) to send `sort` as a simple string, not a `Sort` object with `{ field, direction }`.

## 3. Standardize Course Table Headers

- [x] 3.1 In `bidding-admin/src/components/dashboard/process/bidding-round/course-table-bidding-round.tsx`, change column label `'Seat Available'` → `'Seats Available'` (line ~48) and `'Course'` → `'Course Name'` in the render column (line ~36 label).
- [x] 3.2 In `bidding-admin/src/components/dashboard/process/add-drop/course-table.tsx`, change column label `'Credit'` → `'Credits'` (line ~47) and `'Course'` → `'Course Name'` in the render column (line ~35 label).
- [x] 3.3 In `bidding-admin/src/components/settings/course-table-setting.tsx`, change column label `'Course'` → `'Course Name'` (line ~150).

## 4. Verification

- [ ] 4.1 Test the student list API endpoint with sort params `sort=promotion_name&order=asc`, `sort=programme_name&order=asc`, `sort=home_campus&order=asc` to confirm HTTP 200 with sorted results.
- [ ] 4.2 In the admin UI, navigate to Settings > Students and click each sortable column header (Name, Email, Peoplesoft ID, Promotion, Programme, Credits, Capital Left, Type, Home Campus) to verify sorting works without "No records found!" errors.
- [ ] 4.3 Verify that sorting combined with filters (e.g., filter by promotion + sort by home campus) returns correct filtered and sorted results.
- [ ] 4.4 Verify course table headers display correctly in Bidding Round and Add-Drop dashboard views.
- [ ] 4.5 Smoke test: navigate across all dashboard views that show student and course tables to confirm no regressions.
