# Fix Student List Sort Mapping & Standardize Table Headers

## Problem

1. **Sorting causes "No records found!"**: When clicking sortable column headers (Promotion, Programme, Home Campus) on the student list, the frontend sent sort field names that either didn't match backend expectations or triggered a backend SQL error, resulting in a 500 response displayed as "No records found!".

2. **Sort field mapping mismatches**: The `student-setting-table.tsx` `fieldMapping` sent incorrect backend field names (`full_name` instead of `first_name`, `id` instead of `people_soft_id`, `program` instead of `programme_name`), causing sorts to silently fall back to default.

3. **Sort payload format mismatch**: The settings students page sent sort as a `Sort` object (`{ field: 's.lastName', direction: 'ASC' }`) with DQL paths instead of simple string field names expected by the backend controller.

4. **Inconsistent table headers**: Course tables used different labels across views — "Credit" vs "Credits", "Seat Available" vs "Seats Available", "Course" vs "Course Name".

## Solution

Fixed the frontend sort field mapping, sort payload format, and standardized all course table column headers.

### Changes Made

**Modified Files:**

1. **`src/components/settings/student-setting-table.tsx`**
   - Fixed `fieldMapping` to send correct backend sort field names:
     - `id` → `'people_soft_id'` (was `'id'`)
     - `name` → `'first_name'` (was `'full_name'`)
     - `program` → `'programme_name'` (was `'program'`)

2. **`src/app/(authenticated)/mba/settings/students/page.tsx`**
   - Changed default `currentSort` from `{ sort: 's.lastName', order: 'asc' }` to `{ sort: 'first_name', order: 'asc' }`
   - Fixed `handleSort` to send sort as a simple string field name (not DQL path)
   - Fixed `handleSearch`, `handleClear`, `handleChangePage`, and initial `useEffect` to send `sort` and `order` as separate simple string params instead of a `Sort` object

3. **`src/components/dashboard/process/bidding-round/course-table-bidding-round.tsx`**
   - Changed "Course" → "Course Name"
   - Changed "Seat Available" → "Seats Available"

4. **`src/components/dashboard/process/add-drop/course-table.tsx`**
   - Changed "Course" → "Course Name"
   - Changed "Credit" → "Credits"

5. **`src/components/settings/course-table-setting.tsx`**
   - Changed "Course" → "Course Name"

## Impact

- **No API changes**: Frontend-only fixes — sort params now correctly match backend expectations
- **No breaking changes**: Sort state is ephemeral (React state), not persisted
- **Consistent UX**: All course tables now use uniform column labels across dashboard views

## Verification

- [ ] Navigate to Settings > Students, click each sortable column header — verify sorting works without "No records found!"
- [ ] Verify Name sort sends `sort=first_name`, Peoplesoft ID sends `sort=people_soft_id`
- [ ] Verify Promotion, Programme, Home Campus sorts return sorted results (no 500 error)
- [ ] Verify sorting combined with filters preserves both filter and sort
- [ ] Verify Bidding Round course table shows "Course Name", "Credits", "Seats Available"
- [ ] Verify Add-Drop course table shows "Course Name", "Credits"
- [ ] Verify Settings course table shows "Course Name"
