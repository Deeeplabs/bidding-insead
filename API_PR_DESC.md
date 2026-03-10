# Fix Student List Sorting — SQL Error 3065 (DISTINCT + ORDER BY)

## Problem

When sorting the student list by columns from joined entities (Promotion, Programme, Home Campus), the API returns a 500 error with MySQL 8 error 3065: "Expression #1 of ORDER BY clause is not in SELECT list, references column 'bidding.p2_.label' which is not in SELECT list; this is incompatible with DISTINCT".

**Root cause**: `StudentRepository::queryPaginated()` uses `SELECT DISTINCT s` (Doctrine entity). When `FilterableQueryProvider` applies an `ORDER BY` on a joined column (e.g., `p.label` for Promotion, `c.label` for Campus, `prog.name` for Programme), MySQL 8's strict mode rejects it because the ORDER BY column is not in the SELECT DISTINCT list.

## Solution

Replaced `SELECT DISTINCT` with `GROUP BY` on the primary key, which achieves the same row deduplication without imposing the SELECT-list restriction on ORDER BY columns.

### Changes Made

**Modified File:**

1. **`src/Repository/StudentRepository.php`**
   - **Method**: `queryPaginated(Pagination $pagination, array $filters, Sort $sort)`
   - Replaced `->select("DISTINCT $alias")` with `->select($alias)` + `->groupBy("$alias.id")`
   - `GROUP BY s.id` deduplicates rows (student can appear multiple times due to LEFT JOINs on studentData, campus, etc.) while allowing ORDER BY on any joined column
   - Paginator already uses `fetchJoinCollection: false`, so count query works correctly with GROUP BY

## Impact

- **No API response shape changes**: Same paginated student list response
- **No frontend changes needed**: Backend-only query fix
- **No migration required**: No database schema changes
- **Backward compatible**: `GROUP BY primary_key` is functionally equivalent to `DISTINCT` on the entity

## Verification

- [ ] Sort student list by Promotion (`sort=promotion_name&order=asc`) — should return 200 with sorted results
- [ ] Sort student list by Programme (`sort=programme_name&order=asc`) — should return 200 with sorted results
- [ ] Sort student list by Home Campus (`sort=home_campus&order=asc`) — should return 200 with sorted results
- [ ] Verify pagination totals are correct (no duplicate or missing records)
- [ ] Verify sorting combined with filters returns correct filtered + sorted results
