## Why

The Seat Capacity filter on the Student Switch → Course list popup page (Figma Screen 2-3: Module Modal & All Courses View) is not working correctly. When a student clicks on a module and applies the `seat_min` / `seat_max` filter, the results are inconsistent with the displayed `seat_available` values.

**Root cause:** The `getAvailableCourses()` method in `FlexSwitchService` (student-side) filters on **individual** `ClassPromotions.promotionSeats` rows via `WHERE cp.promotionSeats >= :seatMin`, but the displayed `seat_available` value is the **SUM** of all `ClassPromotions.promotionSeats` for that class (computed in `formatCourses()`). This mismatch means:

- A class with 3 ClassPromotions rows of 30 seats each (total = 90) would pass `seat_max=50` filter because each individual row (30) passes, but displays `seat_available = 90` — misleading the user.
- Conversely, a class might be incorrectly excluded by `seat_min` if one of its individual rows has fewer seats than the minimum, even though the total exceeds the threshold.

**Secondary issue:** The pagination count query uses `COUNT(cl.id)` without `DISTINCT`, inflating counts when a class has multiple `ClassPromotions` rows from the JOIN.

**Technical constraint:** The query uses Doctrine ORM entity hydration (`->select('cl', 'course', ...)`), which returns full entity objects. Doctrine ORM does **not** support `GROUP BY` + `HAVING SUM()` with entity hydration — this pattern only works with scalar/array selects. The fix cannot simply "add GROUP BY and HAVING" to the existing query.

The PM-side `listCourse()` method handles this correctly by using PHP-level filtering after fetching entities: it fetches all classes, computes seat totals via entity methods, and filters in PHP before paginating.

## What Changes

Fix the student-side `getAvailableCourses()` method to:
1. Remove `WHERE` seat filters from the DQL query
2. Apply seat capacity filtering in PHP after fetching entities, using the same total seat sum that `formatCourses()` displays
3. Use `COUNT(DISTINCT cl.id)` for accurate pagination counts (for the non-seat-filter case)
4. For the seat-filter case: fetch all matching classes, filter in PHP by seat sum, then apply pagination manually — matching the PM's `listCourse()` pattern

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
- **Reference implementation:** PM's `listCourse()` in `FlexSwitch\PM\FlexSwitchService.php` (lines 694-717) correctly uses PHP-level seat filtering after entity hydration
