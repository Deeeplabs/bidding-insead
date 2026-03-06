## Context

The student-side `GET /student/flex-switch/courses` endpoint (`FlexSwitchService::getAvailableCourses()`) has a seat capacity filtering bug. The filter compares `seat_min`/`seat_max` against individual `ClassPromotions.promotionSeats` rows instead of the total (SUM) across all ClassPromotions for a class. The displayed `seat_available` in the response correctly sums all ClassPromotions, creating a mismatch between what is filtered and what is shown.

The PM-side `FlexSwitch\PM\FlexSwitchService::listClassConfiguration()` already implements this correctly using `SUM(cp.promotionSeats)` with `GROUP BY` and `HAVING` clauses.

## Goals / Non-Goals

**Goals:**
- Fix seat capacity filter to use SUM of all ClassPromotions seats for a class, matching the displayed `seat_available`
- Fix pagination count to use `COUNT(DISTINCT cl.id)` to avoid inflated counts from JOINs
- Align student-side filtering approach with the proven PM-side pattern

**Non-Goals:**
- No changes to the API response shape
- No changes to other filters (credit, campus, delivery_mode)
- No changes to PM-side endpoints (already correct)
- No database schema changes
- No frontend changes

## Decisions

1. **Refactor query to use GROUP BY + HAVING pattern**: Instead of filtering on `cp.promotionSeats` directly (which operates on individual rows), add `GROUP BY cl.id` and use `HAVING SUM(cp.promotionSeats) >= :seatMin` / `<= :seatMax`. This matches the PM's `listClassConfiguration()` approach (lines 381, 425-432 in `PM\FlexSwitchService.php`).

2. **Fix pagination count**: Change `COUNT(cl.id)` to `COUNT(DISTINCT cl.id)` in the count query clone to avoid inflated totals from JOINed ClassPromotions. For seat filters, use a subquery approach (same as PM's count query at lines 516-531) to correctly filter before counting.

3. **Keep the hard filter `cp.promotionSeats > 0` as-is**: This WHERE clause correctly excludes individual zero-seat ClassPromotions from the JOIN, which is the right behavior (we only want classes that have at least one non-zero seat row).

4. **Approach pattern**: The fix uses the same two-part approach as the PM service:
   - Main query: `GROUP BY cl.id` + `HAVING SUM(...)` for filtering
   - Count query: Subquery with `SUM` for accurate counting with seat filters

## Risks / Trade-offs

- **Low risk**: This is a straightforward fix aligning with an already-proven pattern in the PM service
- **Performance**: Adding `GROUP BY` may slightly increase query complexity, but the PM service already uses this pattern at scale without issues
- **Backward compatibility**: No risk — the API response shape is unchanged, and the filter simply starts working correctly
- **No migration needed**: Pure business logic fix in a single service method
