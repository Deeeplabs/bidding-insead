## Context

The admin dashboard student list sorting triggers MySQL 8 error 3065 when sorting by columns from joined entities (Promotion, Programme, Campus). The root cause is `StudentRepository::queryPaginated()` using `SELECT DISTINCT s` while `FilterableQueryProvider::apply()` adds `ORDER BY` on joined columns not present in the SELECT list. Additionally, the frontend sort field names don't consistently match what the backend `StudentService::listStudents()` expects. Course and student table headers also have minor inconsistencies across dashboard views.

## Goals / Non-Goals

**Goals:**
- Fix the SQL 3065 error so all student list sort columns work without 500 errors
- Align frontend sort field names with backend expected values
- Standardize table column labels across course and student tables

**Non-Goals:**
- Changing the pagination or filter logic
- Adding new sort fields or filter capabilities
- Modifying API response DTOs

## Decisions

### 1. Fix DISTINCT + ORDER BY in StudentRepository

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

### 2. Fix Frontend Sort Field Mapping

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

### 3. Fix Settings Page Sort Payload Format

**Problem**: The settings page (`/mba/settings/students/page.tsx`) `handleSort` stores sort as DQL path strings like `s.lastName` and sends sort as a `Sort` object `{ field: 's.lastName', direction: 'ASC' }`. But when this goes through the `useStudents` hook as query params, the backend `StudentController` expects a simple string for `sort`.

**Fix**: Change `handleSort` in the settings page to send sort as a simple string matching the backend's expected field names (e.g., `first_name`, `promotion_name`), not DQL paths. The default sort should be `first_name` instead of `s.lastName`.

### 4. Standardize Table Headers

**Course tables — changes needed:**
- `course-table-bidding-round.tsx`: "Seat Available" → "Seats Available"
- `course-table.tsx` (add-drop): "Credit" → "Credits"

**Student tables — no changes needed** for core headers. The existing student-setting-table already uses consistent names. Contextual tables (simulation, enrollment) use different columns appropriate to their context and don't need standardization.

## Risks / Trade-offs

- **Risk: GROUP BY changes query performance** → Mitigation: GROUP BY on primary key (`s.id`) is functionally equivalent to DISTINCT on the entity and should not impact performance. MySQL optimizes `GROUP BY primary_key` efficiently.
- **Risk: Paginator behavior with GROUP BY** → Mitigation: The Paginator is already instantiated with `fetchJoinCollection: false`, so the count query will work correctly with GROUP BY.
- **Risk: Changing sort field names breaks existing saved state** → Mitigation: Sort state is ephemeral (in React state), not persisted. Users will simply get a fresh default sort on next page load.
