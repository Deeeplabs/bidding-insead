## 1. Fix Seat Capacity Filter in Student FlexSwitchService

- [x] 1.1 **Remove DQL-level seat filters and apply seat filtering in PHP after entity fetch**
  - File: `bidding-api/src/Service/FlexSwitch/FlexSwitchService.php` — `getAvailableCourses()` method (lines 48-209)
  - **Remove** the `WHERE cp.promotionSeats >= :seatMin` block (lines 102-106)
  - **Remove** the `WHERE cp.promotionSeats <= :seatMax` block (lines 108-112)
  - Keep the existing `WHERE cp.promotionSeats > 0` filter (line 75) — this correctly pre-filters zero-seat rows
  - **Add PHP-level seat filtering** after the query result fetch and before formatting:
    - Check if `seat_min` or `seat_max` filters are present
    - If present: fetch ALL matching classes (skip DB-level pagination), compute total seat sum per class using `$class->getClassPromotions()` loop (same as `formatCourses()`, lines 230-234), filter classes where sum doesn't match criteria, then manually paginate the filtered array with `array_slice()`
    - If not present: keep existing DB-level pagination with `setFirstResult/setMaxResults`
  - Follow the pattern from PM's `listCourse()` in `PM/FlexSwitchService.php` (lines 697-717, 771-772)

- [x] 1.2 **Fix pagination count query to use COUNT(DISTINCT cl.id)**
  - File: `bidding-api/src/Service/FlexSwitch/FlexSwitchService.php` — `getAvailableCourses()` method (lines 139-142)
  - Change `COUNT(cl.id)` to `COUNT(DISTINCT cl.id)` in the count query clone (line 141)
  - This fixes inflated totals when a class has multiple ClassPromotions rows from the JOIN
  - This count is only used in the **non-seat-filter path** (when seat filters are not applied)
  - In the seat-filter path, the total count comes from `count($filteredClasses)` after PHP filtering

## 2. Verification

- [x] 2.1 **Manual API verification**
  - Test `GET /student/flex-switch/courses?seat_min=50` — verify only classes with total seats >= 50 are returned
  - Test `GET /student/flex-switch/courses?seat_max=30` — verify only classes with total seats <= 30 are returned
  - Test `GET /student/flex-switch/courses?seat_min=20&seat_max=60` — verify range filter works correctly
  - Verify `seat_available` in each result matches the filter criteria (the displayed value should always satisfy the filter)
  - Verify `pagination.total_items` is accurate (not inflated)
  - Test without seat filters to confirm no regression in basic course listing
  - Test combined filters (e.g., `seat_min=10&campus_id=1&credit_min=1`) to verify other filters still work
  - Test pagination with seat filters: verify `page=1&per_page=10` with seat filters returns correct page slices
