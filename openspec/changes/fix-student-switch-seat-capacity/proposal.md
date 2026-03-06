## Why

The Seat Capacity filter on the Student Switch → Course list popup page (Figma Screen 2-3: Module Modal & All Courses View) is not working correctly. When a student clicks on a module and applies the `seat_min` / `seat_max` filter, the results are inconsistent with the displayed `seat_available` values.

**Root cause:** The `getAvailableCourses()` method in `FlexSwitchService` (student-side) filters on **individual** `ClassPromotions.promotionSeats` rows (`cp.promotionSeats >= :seatMin`), but the displayed `seat_available` value is the **SUM** of all `ClassPromotions.promotionSeats` for that class. This mismatch means:

- A class with 3 ClassPromotions rows of 30 seats each (total = 90) would pass `seat_max=50` filter because each individual row (30) passes, but displays `seat_available = 90` — misleading the user.
- Conversely, a class might be incorrectly excluded by `seat_min` if one of its individual rows has fewer seats than the minimum, even though the total exceeds the threshold.

**Secondary issue:** The pagination count query uses `COUNT(cl.id)` without `DISTINCT`, inflating counts when a class has multiple `ClassPromotions` rows from the JOIN.

The PM-side `listClassConfiguration()` method already handles this correctly using `SUM(cp.promotionSeats)` with `GROUP BY` and `HAVING` clauses.

## What Changes

Fix the student-side `getAvailableCourses()` method to:
1. Use `HAVING SUM(cp.promotionSeats)` for seat capacity filtering instead of filtering on individual `cp.promotionSeats` rows
2. Use `COUNT(DISTINCT cl.id)` for accurate pagination counts
3. Add proper `GROUP BY` clause to aggregate ClassPromotions correctly

## Capabilities

### New Capabilities
_None — this is a bug fix._

### Modified Capabilities
_No spec-level behavior changes — the API contract remains identical. The filter simply needs to work as documented._

## Impact

- **Affected app:** `bidding-api` (backend only)
- **Affected file:** `bidding-api/src/Service/FlexSwitch/FlexSwitchService.php` — `getAvailableCourses()` method
- **Affected endpoint:** `GET /student/flex-switch/courses` (seat_min, seat_max query parameters)
- **No migration required:** No schema changes needed
- **No API response shape change:** The response format remains identical
- **No breaking changes:** Only the filtering logic is corrected
- **Backward compatible:** Fixes incorrect behavior; no existing correct behavior is altered
- **Reference implementation:** PM's `listClassConfiguration()` in `FlexSwitch\PM\FlexSwitchService.php` (lines 381, 425-432) correctly uses `SUM + GROUP BY + HAVING`
