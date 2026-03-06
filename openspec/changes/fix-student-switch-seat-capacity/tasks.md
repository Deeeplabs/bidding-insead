## 1. Fix Seat Capacity Filter in Student FlexSwitchService

- [x] 1.1 **Refactor `getAvailableCourses()` query to use GROUP BY + HAVING for seat filtering**
  - File: `bidding-api/src/Service/FlexSwitch/FlexSwitchService.php` — `getAvailableCourses()` method (lines 48-209)
  - Replace the current `WHERE cp.promotionSeats >= :seatMin` / `<= :seatMax` (lines 102-112) with `HAVING SUM(cp.promotionSeats)` clauses
  - Add `GROUP BY cl.id` to aggregate ClassPromotions rows per class
  - Keep the existing `WHERE cp.promotionSeats > 0` filter (line 75) as-is — this correctly filters out zero-seat rows before aggregation
  - Follow the pattern from PM's `listClassConfiguration()` in `bidding-api/src/Service/FlexSwitch/PM/FlexSwitchService.php` (lines 381, 392, 425-432)

- [x] 1.2 **Fix pagination count query to use COUNT(DISTINCT)**
  - File: `bidding-api/src/Service/FlexSwitch/FlexSwitchService.php` — `getAvailableCourses()` method (lines 139-142)
  - Change `COUNT(cl.id)` to `COUNT(DISTINCT cl.id)` in the count query clone
  - For seat filter case: build a separate count query using subquery with SUM (follow PM pattern at lines 516-531 in `PM/FlexSwitchService.php`)

## 2. Verification

- [x] 2.1 **Manual API verification**
  - Test `GET /student/flex-switch/courses?seat_min=50` — verify only classes with total seats >= 50 are returned
  - Test `GET /student/flex-switch/courses?seat_max=30` — verify only classes with total seats <= 30 are returned
  - Test `GET /student/flex-switch/courses?seat_min=20&seat_max=60` — verify range filter works correctly
  - Verify `seat_available` in each result matches the filter criteria
  - Verify `pagination.total_items` is accurate (not inflated)
  - Test without seat filters to confirm no regression in basic course listing
