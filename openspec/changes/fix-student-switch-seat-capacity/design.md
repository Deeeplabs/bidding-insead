## Context

The student-side `GET /student/flex-switch/courses` endpoint (`FlexSwitchService::getAvailableCourses()`) has a seat capacity filtering bug. The filter compares `seat_min`/`seat_max` against individual `ClassPromotions.promotionSeats` rows via `WHERE` clauses, but the displayed `seat_available` in the response is the SUM of all ClassPromotions for that class (computed in `formatCourses()` via a PHP loop). This creates a mismatch between what is filtered and what is shown.

**Critical technical constraint:** The query uses Doctrine ORM **entity hydration** — `->select('cl', 'course', 'campus', 'cp', 'promotion', 'courseType', 'pp')` — which returns full entity objects. Doctrine cannot use `GROUP BY` + `HAVING SUM()` with entity-level selects. The `GROUP BY` + `HAVING` pattern only works with scalar/array selects (e.g., `->select('cl.id', 'SUM(cp.promotionSeats) as totalSeats')`).

The PM-side has two approaches:
1. `listClassConfiguration()` — uses scalar selects with `GROUP BY` + `HAVING SUM()` (lines 396-410, 443-450)
2. `listCourse()` — uses entity hydration with `GROUP BY cl.id`, then filters seats in PHP (lines 671, 700-717)

Since the student-side `getAvailableCourses()` uses entity hydration and `formatCourses()` depends on entity methods, the correct approach is to follow the PM's `listCourse()` pattern: **PHP-level seat filtering**.

## Goals / Non-Goals

**Goals:**
- Fix seat capacity filter to use the total SUM of all ClassPromotions seats for a class, matching the displayed `seat_available`
- Fix pagination count to use `COUNT(DISTINCT cl.id)` to avoid inflated counts from JOINs
- Handle seat-filtered pagination correctly (fetch all, filter in PHP, slice for page)
- Align student-side filtering approach with the proven PM-side `listCourse()` pattern

**Non-Goals:**
- No changes to the API response shape
- No refactoring the query to use scalar selects (would require rewriting `formatCourses()`)
- No changes to other filters (credit, campus, delivery_mode, search, promotion_period)
- No changes to PM-side endpoints (already correct)
- No database schema changes
- No frontend changes

## Decisions

1. **Remove seat filters from DQL, apply in PHP**: Remove the `WHERE cp.promotionSeats >= :seatMin` and `WHERE cp.promotionSeats <= :seatMax` clauses from the query builder. Instead, after fetching entities, compute the total seat sum per class (same logic as `formatCourses()`) and filter before pagination. This exactly matches PM's `listCourse()` pattern (lines 700-717).

2. **Two pagination paths**: 
   - **Without seat filters**: Use the existing `COUNT(DISTINCT cl.id)` approach for efficient DB-level pagination (clone query, count, then paginate with `setFirstResult/setMaxResults`).
   - **With seat filters**: Fetch all matching classes (no pagination at DB level), compute seat sums, filter in PHP, then slice the result array for the requested page. This is necessary because the seat filter cannot be expressed in DQL with entity hydration.

3. **Keep the hard filter `cp.promotionSeats > 0` as-is**: This WHERE clause correctly excludes classes where ALL ClassPromotions have zero seats. It's a pre-filter that reduces the result set before PHP-level processing.

4. **Fix count query to use `COUNT(DISTINCT cl.id)`**: The current `COUNT(cl.id)` inflates the count when a class has multiple ClassPromotions rows. Change to `COUNT(DISTINCT cl.id)` for accurate pagination in the non-seat-filter path.

## Risks / Trade-offs

- **Performance concern for seat-filtered queries**: When seat filters are applied, ALL matching classes must be fetched from DB before PHP filtering. This could be slower for very large datasets. However, the PM's `listCourse()` already uses this exact pattern at production scale without issues. The dataset is pre-filtered by student's promotion, course type, and `cp.promotionSeats > 0`, which limits the result set significantly.
- **Low risk overall**: This is a straightforward fix aligning with an already-proven PM-side pattern.
- **Backward compatibility**: No risk — the API response shape is unchanged, and the filter simply starts working correctly.
- **No migration needed**: Pure business logic fix in a single service method.
