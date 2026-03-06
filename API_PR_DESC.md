# Fix Student Switch — Seat Capacity Filter Not Working Correctly

## Problem

When a Student navigates to **Switch → clicks on a module → Course list popup** (Figma Screen 2-3), the **Seat Capacity** filter (`seat_min` / `seat_max`) does not work correctly. The filter produces inconsistent results — classes that should be excluded by the filter still appear, and vice versa.

**Root cause:** The `getAvailableCourses()` method in `FlexSwitchService` (student-side) filters on **individual** `ClassPromotions.promotionSeats` rows using `WHERE cp.promotionSeats >= :seatMin`, but the displayed `seat_available` value in the response is the **SUM** of all `ClassPromotions.promotionSeats` for that class.

**Example:** A class with 3 ClassPromotions rows of 30 seats each (total = 90) would:
- ❌ Pass `seat_max=50` filter (because each individual row 30 ≤ 50) — but shows `seat_available = 90`
- ❌ Incorrectly match the filter while displaying a value that contradicts it

**Secondary issue:** The pagination count query uses `COUNT(cl.id)` without `DISTINCT`, inflating total counts when a class has multiple `ClassPromotions` rows from the JOIN.

> The PM-side `listClassConfiguration()` already handles this correctly using `SUM(cp.promotionSeats)` with `GROUP BY` and `HAVING`.

## Solution

Single file fix in the API backend — no frontend changes, no migration required:

### Changes Made

**Modified File:**

1. **`src/Service/FlexSwitch/FlexSwitchService.php`**
   - **Method**: `getAvailableCourses(Student $student, array $filters)`

   **Seat filter fix:**
   - Removed incorrect `WHERE cp.promotionSeats >= :seatMin` / `<= :seatMax` clauses
   - Added `GROUP BY cl.id` to aggregate ClassPromotions per class
   - Added `HAVING SUM(cp.promotionSeats) >= :seatMin` / `<= :seatMax` to filter on total seats
   - This ensures the filter matches the displayed `seat_available` value

   **Pagination count fix:**
   - Replaced `clone $qb` + `COUNT(cl.id)` with a dedicated count query using `COUNT(DISTINCT cl2.id)`
   - All WHERE conditions are replicated on the count query
   - Seat filter for count uses subquery `(SELECT SUM(cp.promotionSeats) ... WHERE cp.class = cl.id) >= :seatMin` pattern
   - Follows the same approach as PM's `listClassConfiguration()` count query

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
