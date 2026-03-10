# Fix SQL Errors on Student & Course List Sorting

## Problems

### 1. Course list sorting — SQL Error 1055 (only_full_group_by)

`GET /v2/api/courses?page=1&limit=10&sort=credits&order=ASC` returns a 500 error:

> SQLSTATE[42000]: Syntax error or access violation: 1055 Expression #19 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'bidding.c2_.id' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by

**Root cause**: `ClassesRepository::searchQueryPaginated()` in campaign group mode uses `GROUP BY cl.id, c.id, s.id, ct.id, cam.id, stat.id` but has LEFT JOINs to `classPromotions (cp)`, `promotion (p)`, `promotionPeriods (pp)`. The Doctrine Paginator with `fetchJoinCollection: true` internally wraps the query in a subquery whose SELECT includes columns from those LEFT JOINed tables — columns not covered by the GROUP BY — causing the MySQL violation.

### 2. Student list sorting — SQL Error 3065 (DISTINCT + ORDER BY)

Sorting the student list by joined entity columns (Promotion, Programme, Home Campus) returns a 500 error with MySQL 8 error 3065: "Expression #1 of ORDER BY clause is not in SELECT list … incompatible with DISTINCT".

**Root cause**: `StudentRepository::queryPaginated()` uses `SELECT DISTINCT s`. When `FilterableQueryProvider` applies `ORDER BY` on a joined column (e.g., `p.label`), MySQL rejects it because the ORDER BY column is not in the SELECT DISTINCT list.

## Solution

### 1. Course list — Disable `fetchJoinCollection` in campaign group mode

Set `fetchJoinCollection: !$campaignGroupMode` on the Doctrine Paginator. When campaign group mode is active, `GROUP BY cl.id` already ensures one row per class, making the Paginator's subquery-wrapping (which causes the column conflict) unnecessary. The total count is already computed separately via `COUNT(DISTINCT cl.id)` and passed as `totalOverride`.

### 2. Student list — Replace `DISTINCT` with `GROUP BY`

Replaced `SELECT DISTINCT` with `GROUP BY` on the primary key, which achieves the same row deduplication without imposing the SELECT-list restriction on ORDER BY columns.

## Changes Made

1. **`src/Repository/ClassesRepository.php`**
   - **Method**: `searchQueryPaginated()`
   - Changed Paginator construction from `fetchJoinCollection: true` to `fetchJoinCollection: !$campaignGroupMode`
   - When `campaignGroupMode` is true (set by `/v2/api/courses` endpoint), the Paginator no longer wraps the query in a subquery, avoiding the GROUP BY / only_full_group_by conflict

2. **`src/Repository/StudentRepository.php`**
   - **Method**: `queryPaginated()`
   - Replaced `->select("DISTINCT $alias")` with `->select($alias)` + `->groupBy("$alias.id")`
   - `GROUP BY s.id` deduplicates rows while allowing ORDER BY on any joined column
   - Paginator already uses `fetchJoinCollection: false`, so count query works correctly

## Impact

- **No API response shape changes**: Same paginated responses for both endpoints
- **No migration required**: No database schema changes
- **Backward compatible**: Both fixes are functionally equivalent to the previous behavior — only the SQL generation strategy changes
- **Scoped**: The `fetchJoinCollection` change only affects the campaign group mode path; non-campaign queries retain `fetchJoinCollection: true`

## Verification

- [ ] `GET /v2/api/courses?page=1&limit=10&sort=credits&order=ASC` — returns 200 with sorted courses (no SQL 1055)
- [ ] `GET /v2/api/courses?page=1&limit=10&sort=name&order=ASC` — returns 200
- [ ] `GET /v2/api/courses?page=1&limit=10&sort=type&order=DESC` — returns 200
- [ ] `GET /v2/api/courses?page=1&limit=10` (no sort) — returns 200 with default ordering
- [ ] Sort student list by Promotion (`sort=promotion_name&order=asc`) — returns 200
- [ ] Sort student list by Programme (`sort=programme_name&order=asc`) — returns 200
- [ ] Sort student list by Home Campus (`sort=home_campus&order=asc`) — returns 200
- [ ] Pagination totals are correct for both endpoints
- [ ] Sorting combined with filters returns correct results
