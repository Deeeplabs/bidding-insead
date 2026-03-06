# Fix Student Switch — Seat Capacity Filter Not Working Correctly

## Problem

When a Student navigates to **Switch → clicks on a module → Course list popup** (Figma Screen 2-3), the **Seat Capacity** filter (`seat_min` / `seat_max`) does not work correctly. The filter produces inconsistent results — classes that should be excluded by the filter still appear, and vice versa.

**Root cause:** The `getAvailableCourses()` method in `FlexSwitchService` (student-side) filters on **individual** `ClassPromotions.promotionSeats` rows using `WHERE cp.promotionSeats >= :seatMin`, but the displayed `seat_available` value in the response is the **SUM** of all `ClassPromotions.promotionSeats` for that class (computed in `formatCourses()`).

**Example:** A class with 3 ClassPromotions rows of 30 seats each (total = 90) would:
- ❌ Pass `seat_max=50` filter (because each individual row 30 ≤ 50) — but shows `seat_available = 90`
- ❌ Incorrectly match the filter while displaying a value that contradicts it

**Secondary issue:** The pagination count query uses `COUNT(cl.id)` without `DISTINCT`, inflating total counts when a class has multiple `ClassPromotions` rows from the JOIN.

> The PM-side `listCourse()` already handles this correctly using PHP-level seat filtering after entity hydration.

## Solution

Single file fix in the API backend — no frontend changes, no migration required.

### Changes Made

**Modified File:**

1. **`src/Service/FlexSwitch/FlexSwitchService.php`**
   - **Method**: `getAvailableCourses(Student $student, array $filters)`

   **Seat filter fix:**
   - Removed incorrect `WHERE cp.promotionSeats >= :seatMin` / `<= :seatMax` DQL clauses
   - Seat filtering is now done **in PHP** after fetching entities — computes the total SUM of all `ClassPromotions.promotionSeats` per class, then filters against `seat_min`/`seat_max`
   - This ensures the filter matches the exact same `seat_available` value displayed in the response
   - Follows the PM's `listCourse()` pattern (PHP-level filtering) — `GROUP BY + HAVING SUM()` cannot be used here because the query uses Doctrine entity hydration

   **Two execution paths:**
   - **Without seat filters**: Standard DB-level pagination with `COUNT(DISTINCT cl.id)` — efficient, no behavior change
   - **With seat filters**: Fetches all matching classes → filters by seat sum in PHP → paginates with `array_slice()` — correct results

   **Pagination count fix:**
   - Changed `COUNT(cl.id)` to `COUNT(DISTINCT cl.id)` to avoid inflated totals from JOINed ClassPromotions rows
   - When seat filters are active, total count comes from `count($filteredClasses)` after PHP filtering

## Impact

- **No API response shape changes**: The response format remains identical — only the filtering logic is corrected
- **No frontend changes needed**: `bidding-web` Student Switch UI consumes the same response format
- **No migration required**: No database schema changes
- **No breaking changes**: Fixes incorrect behavior; no existing correct behavior is altered
- **Backward compatible**: Pure business logic fix

## Verification

- [ ] Student: Switch → Click module → Apply seat_min filter → Only classes with total seats ≥ min shown
- [ ] Student: Switch → Click module → Apply seat_max filter → Only classes with total seats ≤ max shown
- [ ] Student: Switch → Click module → Apply seat range → Only classes within range shown
- [ ] `seat_available` value in each result matches the filter criteria
- [ ] `pagination.total_items` is accurate (not inflated by JOINs)
- [ ] No regression in basic course listing without seat filters
