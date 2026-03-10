## Context

Two distinct SQL errors occur in the admin dashboard:

1. **Course list sorting** — `GET /v2/api/courses?sort=credits&order=ASC` triggers MySQL error 1055 (only_full_group_by). The endpoint calls `CourseController::filterCourses()` → `ClassService::getList()` → `ClassesRepository::searchQueryPaginated()`. When `_campaign_group_mode` is active, the query uses `GROUP BY cl.id, c.id, s.id, ct.id, cam.id, stat.id` but has LEFT JOINs to `classPromotions (cp)`, `promotion (p)`, `promotionPeriods (pp)`. The Doctrine Paginator with `fetchJoinCollection: true` internally generates a wrapping subquery whose SELECT includes columns from those LEFT JOINed tables — columns not covered by the GROUP BY — causing the MySQL violation.

2. **Student list sorting** — Sorting by joined entity columns (Promotion, Programme, Campus) triggers MySQL error 3065 (DISTINCT + ORDER BY). `StudentRepository::queryPaginated()` uses `SELECT DISTINCT s`; when `FilterableQueryProvider` adds ORDER BY on a joined column (e.g., `p.label`), MySQL rejects it.

Additionally, frontend sort field names don't match backend expectations, and course/student table headers have minor inconsistencies.

## Goals / Non-Goals

**Goals:**
- Fix the SQL 1055 error on the courses endpoint so sorting by any field works
- Fix the SQL 3065 error so all student list sort columns work without 500 errors
- Align frontend sort field names with backend expected values
- Standardize table column labels across course and student tables

**Non-Goals:**
- Changing the pagination or filter logic beyond what's needed to fix the SQL errors
- Adding new sort fields or filter capabilities
- Modifying API response DTOs

## Decisions

### 1. Fix GROUP BY + Paginator conflict in ClassesRepository

**Problem**: `ClassesRepository::searchQueryPaginated()` (line ~697) in campaign group mode uses:
```php
$query->groupBy('cl.id, c.id, s.id, ct.id, cam.id, stat.id');
```
The query also has LEFT JOINs to `cp` (classPromotions), `p` (promotion), `pp` (promotionPeriods). The Doctrine Paginator is created with `fetchJoinCollection: true` (line ~736):
```php
new Paginator(query: $query, fetchJoinCollection: true)
```
When `fetchJoinCollection` is true, the Paginator uses `LimitSubqueryOutputWalker` to wrap the query in a subquery for correct pagination with JOINs. This walker expands the SELECT list to include columns from LEFT JOINed entities. Those columns are not in the GROUP BY, so MySQL's `only_full_group_by` mode rejects the query.

**Fix**: When `campaignGroupMode` is true, set `fetchJoinCollection` to `false`. The `GROUP BY cl.id` already ensures one row per class, so the Paginator's subquery-wrapping is unnecessary. The count is already computed separately via `totalOverride`, so the Paginator's count behavior isn't affected.

```php
// Change the Paginator construction to be conditional:
return new PaginatedResult(
    paginator: new Paginator(
        query: $query,
        fetchJoinCollection: !$campaignGroupMode  // false when campaign mode
    ),
    limit: $pagination->limit,
    offset: $pagination->offset,
    totalOverride: $totalCount,
);
```

**Why this is safe**: 
- `GROUP BY cl.id` already deduplicates — `fetchJoinCollection`'s dedup subquery is redundant.
- The count is already overridden by `$totalCount` (computed via a separate `COUNT(DISTINCT cl.id)` query), so the Paginator's internal count is not used.
- The `$campaignGroupMode` flag is only set by `CourseController::filterCourses()`, which always provides the `_campaign_group_mode` filter.

**Alternative considered**: Adding `cp.id, p.id, pp.id` to the campaign GROUP BY. Rejected because the comment in the code explicitly states these are omitted "to ensure class count matches campaign detail's total_sections". Adding them back would break the intended behavior.

### 2. Fix DISTINCT + ORDER BY in StudentRepository

**Problem**: `StudentRepository::queryPaginated()` at line 131 uses:
```php
->select("DISTINCT $alias")
```
When `FilterableQueryProvider::apply()` adds an ORDER BY on a joined column (e.g., `p.label`, `c.label`, `prog.name`), Doctrine generates SQL like:
```sql
SELECT DISTINCT s0_.* FROM student s0_ ... ORDER BY p2_.label ASC
```
MySQL 8 rejects this because `p2_.label` is not in the SELECT DISTINCT list.

**Fix**: Replace `SELECT DISTINCT s` with a `GROUP BY s.id` approach. This eliminates the DISTINCT keyword while still deduplicating rows (student can appear multiple times due to LEFT JOINs on studentData, campus, etc.). The ORDER BY on joined columns is then valid because GROUP BY doesn't impose the same SELECT-list restriction that DISTINCT does.

```php
// Before:
->select("DISTINCT $alias")

// After:
->select($alias)
->groupBy("$alias.id")
```

**Alternative considered**: Adding joined columns to SELECT via `->addSelect('p.label', 'c.label', 'prog.name')`. Rejected because this changes the query result from entity objects to mixed arrays, breaking the existing Paginator and entity mapping.

### 3. Fix Frontend Sort Field Mapping

**Problem**: `student-setting-table.tsx` `fieldMapping` sends field names that don't match what `StudentService::listStudents()` expects:

| Frontend sends | Backend expects | Status |
|---|---|---|
| `full_name` | `first_name` | ❌ Mismatch → falls to default |
| `id` | `people_soft_id` | ❌ Mismatch → falls to default |
| `programme_name` | `programme_name` | ✅ Match but triggers SQL error |
| `promotion_name` | `promotion_name` | ✅ Match but triggers SQL error |
| `home_campus` | `home_campus` | ✅ Match but triggers SQL error |
| `program` | N/A | ❌ Not handled → falls to default |

**Fix**: Update `fieldMapping` in `student-setting-table.tsx`:
```ts
const fieldMapping: Record<SortField, string> = {
  id: 'people_soft_id',        // was 'id'
  name: 'first_name',          // was 'full_name'
  email: 'email',              // OK
  class: 'programme_name',     // OK
  credits: 'credits',          // OK
  capital: 'capital_left',     // OK
  type: 'student_type',        // OK
  home_campus: 'home_campus',  // OK
  promotion_name: 'promotion_name', // OK
  program: 'programme_name',   // was 'program' (unmapped)
};
```

### 4. Fix Settings Page Sort Payload Format

**Problem**: The settings page (`/mba/settings/students/page.tsx`) `handleSort` stores sort as DQL path strings like `s.lastName` and sends sort as a `Sort` object `{ field: 's.lastName', direction: 'ASC' }`. But when this goes through the `useStudents` hook as query params, the backend `StudentController` expects a simple string for `sort`.

**Fix**: Change `handleSort` in the settings page to send sort as a simple string matching the backend's expected field names (e.g., `first_name`, `promotion_name`), not DQL paths. The default sort should be `first_name` instead of `s.lastName`.

### 5. Standardize Table Headers

**Course tables — changes needed:**
- `course-table-bidding-round.tsx`: "Seat Available" → "Seats Available"
- `course-table.tsx` (add-drop): "Credit" → "Credits"

**Student tables — no changes needed** for core headers. The existing student-setting-table already uses consistent names. Contextual tables (simulation, enrollment) use different columns appropriate to their context and don't need standardization.

## Risks / Trade-offs

- **Risk: `fetchJoinCollection: false` causes incorrect pagination** → Mitigation: The `GROUP BY cl.id` already ensures one row per class, making the Paginator's subquery-based dedup unnecessary. The count is already overridden via `totalOverride`. Tested: the non-campaign mode path with `fetchJoinCollection: true` remains untouched.
- **Risk: GROUP BY changes query performance in StudentRepository** → Mitigation: GROUP BY on primary key (`s.id`) is functionally equivalent to DISTINCT on the entity and should not impact performance. MySQL optimizes `GROUP BY primary_key` efficiently.
- **Risk: Paginator behavior with GROUP BY in StudentRepository** → Mitigation: The Paginator is already instantiated with `fetchJoinCollection: false`, so the count query will work correctly with GROUP BY.
- **Risk: Changing sort field names breaks existing saved state** → Mitigation: Sort state is ephemeral (in React state), not persisted. Users will simply get a fresh default sort on next page load.
