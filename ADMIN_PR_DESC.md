# Fix Student List Sort Mapping & Standardize Table Headers + Course List Computed Column Sorting

## Problem

1. **Sorting causes "No records found!"**: When clicking sortable column headers (Promotion, Programme, Home Campus) on the student list, the frontend sent sort field names that either didn't match backend expectations or triggered a backend SQL error, resulting in a 500 response displayed as "No records found!".

2. **Sort field mapping mismatches**: The `student-setting-table.tsx` `fieldMapping` sent incorrect backend field names (`full_name` instead of `first_name`, `id` instead of `people_soft_id`, `program` instead of `programme_name`), causing sorts to silently fall back to default.

3. **Sort payload format mismatch**: The settings students page sent sort as a `Sort` object (`{ field: 's.lastName', direction: 'ASC' }`) with DQL paths instead of simple string field names expected by the backend controller.

4. **Inconsistent table headers**: Course tables used different labels across views — "Credit" vs "Credits", "Seat Available" vs "Seats Available", "Course" vs "Course Name".

5. **Course list columns not sortable**: The Seats, Conflicts, Fallbacks, and Promotions columns in the course table (`course-table-setting.tsx`) were not clickable for sorting. Only Name, Credits, and Type had sort functionality.

## Solution

Fixed the frontend sort field mapping, sort payload format, standardized all course table column headers, and added sorting support for Seats, Conflicts, Fallbacks, and Promotions columns.

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
   - Added `seat`, `conflict`, `fallback`, `promotions` to `SortField` type union
   - Added these 4 fields to `fieldMapping` and `backendSortableFields` arrays
   - Made Seats, Conflicts, Fallbacks, Promotions column headers clickable with `cursor-pointer hover:bg-gray-200`, `onClick` sort handler, and sort direction icon (↑↓↕)
   - Added sort cases in `sortedCourses` `useMemo` for client-side fallback sorting (numeric comparison for seat/conflict/fallback, string comparison for promotions)

## Impact

- **No API changes**: Frontend-only fixes — sort params now correctly match backend expectations
- **No breaking changes**: Sort state is ephemeral (React state), not persisted
- **Consistent UX**: All course tables now use uniform column labels across dashboard views
- **New sort columns**: Seats, Conflicts, Fallbacks, Promotions are now sortable in the Settings course table, sending `sort=seat|conflict|fallback|promotions` to the backend which handles in-memory sorting for these computed fields

## Verification

- [ ] Navigate to Settings > Students, click each sortable column header — verify sorting works without "No records found!"
- [ ] Verify Name sort sends `sort=first_name`, Peoplesoft ID sends `sort=people_soft_id`
- [ ] Verify Promotion, Programme, Home Campus sorts return sorted results (no 500 error)
- [ ] Verify sorting combined with filters preserves both filter and sort
- [ ] Verify Bidding Round course table shows "Course Name", "Credits", "Seats Available"
- [ ] Verify Add-Drop course table shows "Course Name", "Credits"
- [ ] Verify Settings course table shows "Course Name"
- [ ] Click Seats column header in Settings > Courses — sort icon toggles and data re-sorts by available seats
- [ ] Click Conflicts column header — sort icon toggles and data re-sorts by conflict count
- [ ] Click Fallbacks column header — sort icon toggles and data re-sorts
- [ ] Click Promotions column header — sort icon toggles and data re-sorts alphabetically by promotion labels
- [ ] Verify sort toggles cycle: ASC (↑) → DESC (↓) → clear (↕)
