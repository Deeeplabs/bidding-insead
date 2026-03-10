## Context

Two distinct SQL errors occur in the admin dashboard:

1. **Course list sorting** — `GET /v2/api/courses?sort=credits&order=ASC` triggers MySQL error 1055 (only_full_group_by). The endpoint calls `CourseController::filterCourses()` → `ClassService::getList()` → `ClassesRepository::searchQueryPaginated()`. When `_campaign_group_mode` is active, the query uses `GROUP BY cl.id, c.id, s.id, ct.id, cam.id, stat.id` but has LEFT JOINs to `classPromotions (cp)`, `promotion (p)`, `promotionPeriods (pp)`. The Doctrine Paginator with `fetchJoinCollection: true` internally generates a wrapping subquery whose SELECT includes columns from those LEFT JOINed tables — columns not covered by the GROUP BY — causing the MySQL violation.

2. **Student list sorting** — Sorting by joined entity columns (Promotion, Programme, Campus) triggers MySQL error 3065 (DISTINCT + ORDER BY). `StudentRepository::queryPaginated()` uses `SELECT DISTINCT s`; when `FilterableQueryProvider` adds ORDER BY on a joined column (e.g., `p.label`), MySQL rejects it.

Additionally, frontend sort field names don't match backend expectations, and course/student table headers have minor inconsistencies.

3. **Course sort field mapping incomplete** — Sorting by `course_class_section` triggers `Field "course_class_section" is not allowed for sorting`. `CourseController::filterCourses()` `$sortFieldMap` (line ~315) does not include `course_class_section`, so the value passes through unmapped to the validator. `CourseListQueryValidator::sortConstraints()` only allows `c.name`, `c.id`, `c.credit` — missing `cl.section` (for section) and `ct.name` (for type).

4. **Missing `class_id` in course list response** — The `filterCourses()` endpoint iterates over Class entities and maps them to `CourseDto`, but the class entity's ID is not included in the response. This makes it impossible for the frontend to uniquely identify class sections.

## Goals / Non-Goals

**Goals:**
- Fix the SQL 1055 error on the courses endpoint so sorting by any field works
- Fix the SQL 3065 error so all student list sort columns work without 500 errors
- Align frontend sort field names with backend expected values
- Standardize table column labels across course and student tables
- Fix course sort field mapping so `course_class_section` and `type` sorting work
- Add `class_id` to the course list response
- Add sorting support for computed columns (`seat`, `conflict`, `fallback`, `promotions`) via in-memory sorting

**Non-Goals:**
- Changing the pagination or filter logic beyond what's needed to fix the SQL errors and support computed-column sorting
- Adding new filter capabilities beyond what already exists
- Modifying API response DTOs beyond adding `class_id`

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

### 5. Fix Course Sort Field Mapping

**Problem**: `CourseController::filterCourses()` (line ~315) has a `$sortFieldMap`:
```php
$sortFieldMap = [
    'name' => 'c.name',
    'id' => 'c.id',
    'credits' => 'c.credit',
    'credit' => 'c.credit',
    'type' => 'ct.name',
];
$field = $sortFieldMap[$request->sort] ?? $request->sort;
```

`course_class_section` is not in the map, so it passes through as-is. Then `CourseListQueryValidator::sortConstraints()` rejects it because only `c.name`, `c.id`, `c.credit` are allowed. Additionally, `ct.name` (for type sorting) is in the `$sortFieldMap` but NOT in the validator, so type sorting would also fail.

**Fix**:
1. Add `'course_class_section' => 'cl.section'` to `$sortFieldMap` in `CourseController::filterCourses()`
2. Add `SortValidationConstraint::any('cl.section')` and `SortValidationConstraint::any('ct.name')` to `CourseListQueryValidator::sortConstraints()`

```php
// CourseController::filterCourses() $sortFieldMap:
$sortFieldMap = [
    'name' => 'c.name',
    'id' => 'c.id',
    'credits' => 'c.credit',
    'credit' => 'c.credit',
    'type' => 'ct.name',
    'course_class_section' => 'cl.section',
];

// CourseListQueryValidator::sortConstraints():
public function sortConstraints(): array
{
    return [
        SortValidationConstraint::any('c.name'),
        SortValidationConstraint::any('c.id'),
        SortValidationConstraint::any('c.credit'),
        SortValidationConstraint::any('ct.name'),
        SortValidationConstraint::any('cl.section'),
    ];
}
```

**Why this is safe**: The `cl` alias refers to the Class entity which is the root entity in `ClassesRepository::searchQueryPaginated()`. The `ct` alias is for CourseType, always joined. Both columns are safe to ORDER BY.

### 6. Add `class_id` to Course List Response

**Problem**: `CourseController::filterCourses()` iterates over Class entities but only maps the parent Course to `CourseDto`. The class entity's ID is not exposed, making it impossible for the frontend to uniquely identify class sections.

**Fix**:
1. Add `public ?int $class_id = null;` to `CourseDto`
2. In `CourseController::filterCourses()`, set `$courseDto->class_id = $class->getId();` after mapping
3. Add `class_id: number;` to `SingleCourse` type in `course-response.ts`

```php
// CourseDto.php — add property:
#[OA\Property(type: 'integer', example: 42, nullable: true, description: 'Class section ID')]
public ?int $class_id = null;

// CourseController::filterCourses() — after $courseDto = $this->mapper->map(...):
$courseDto->class_id = $class->getId();
```

**Why this is safe**: Additive change — no existing consumers are affected. The `class_id` is simply the primary key of the Class entity that is already being iterated.

### 7. Standardize Table Headers

**Course tables — changes needed:**
- `course-table-bidding-round.tsx`: "Seat Available" → "Seats Available"
- `course-table.tsx` (add-drop): "Credit" → "Credits"

**Student tables — no changes needed** for core headers. The existing student-setting-table already uses consistent names. Contextual tables (simulation, enrollment) use different columns appropriate to their context and don't need standardization.

### 8. Add In-Memory Sorting for Computed Columns (seat, conflict, fallback, promotions)

**Problem**: The course list table has four columns that are computed in PHP after the SQL query returns results:
- `seat` = `max(0, promotionSeats - enrolledCount - invitedCount - waitlistedCount)` — requires `BidService` calls per class
- `conflict` = count of comma-separated IDs from `$class->getClassConflicts()`
- `fallback` = hardcoded to `0`
- `promotions` = comma-separated promotion labels from `ClassPromotions`

These values are NOT database columns and cannot be sorted at the SQL/DQL level. The frontend `course-table-setting.tsx` currently marks only `name`, `credits`, and `type` as sortable — `seat`, `conflict`, `fallback`, and `promotions` columns have no sort capability.

**Fix**: Implement a two-phase approach in `CourseController::filterCourses()`:

1. **Detect computed-field sort**: Define a list of computed sort fields (`seat`, `conflict`, `fallback`, `promotions`). When `$request->sort` matches one of these, skip SQL-level sorting and fetch ALL records (override pagination to fetch everything).

2. **Build DTOs, sort in PHP, then paginate**:
   - Iterate over ALL classes from the query result (no SQL LIMIT/OFFSET when sorting by computed field)
   - Compute `seat`, `conflict`, `fallback`, `promotions` for each `CourseDto` (same logic as existing loop)
   - Sort the full `$response->items` array in PHP using `usort()` based on the requested field and direction
   - Slice the sorted array for the requested page: `array_slice($items, $offset, $limit)`
   - Build correct pagination metadata from the total count

```php
// In CourseController::filterCourses(), before building the query:
$computedSortFields = ['seat', 'conflict', 'fallback', 'promotions'];
$isComputedSort = $request->sort !== null && in_array($request->sort, $computedSortFields, true);

// When $isComputedSort is true:
// 1. Don't pass Sort to classService->getList() (no SQL ORDER BY)
// 2. Use a large limit (10000) and offset 0 to fetch all records
// 3. After building all DTOs, sort in PHP:
if ($isComputedSort) {
    usort($response->items, function (CourseDto $a, CourseDto $b) use ($request) {
        $aVal = $a->{$request->sort};
        $bVal = $b->{$request->sort};
        // Handle null values (push to end)
        if ($aVal === null && $bVal === null) return 0;
        if ($aVal === null) return 1;
        if ($bVal === null) return -1;
        // For string fields (promotions), compare as strings
        if (is_string($aVal)) {
            $cmp = strcasecmp($aVal, $bVal);
        } else {
            $cmp = $aVal <=> $bVal;
        }
        return strtoupper($request->order ?? 'ASC') === 'DESC' ? -$cmp : $cmp;
    });
    // Slice for pagination
    $totalRecords = count($response->items);
    $page = $request->page > 0 ? $request->page : 1;
    $limit = $request->limit > 0 ? $request->limit : 20;
    $offset = ($page - 1) * $limit;
    $response->items = array_slice($response->items, $offset, $limit);
    $response->pagination = new Pagination(
        totalRecords: $totalRecords,
        currentPage: $page,
        totalPages: (int) ceil($totalRecords / $limit),
    );
}
```

**Frontend changes** in `course-table-setting.tsx`:
- Add `seat`, `conflict`, `fallback`, `promotions` to `SortField` type
- Add them to `fieldMapping` and `backendSortableFields`
- Add click handlers and sort icons to the Seats, Conflicts, Fallbacks, Promotions column headers
- Add sort cases in `sortedCourses` for these fields (as fallback for client-side sort)

```ts
type SortField = 'name' | 'credits' | 'type' | 'seat' | 'conflict' | 'fallback' | 'promotions';

const fieldMapping: Record<SortField, string> = {
  name: 'name',
  credits: 'credits',
  type: 'type',
  seat: 'seat',
  conflict: 'conflict',
  fallback: 'fallback',
  promotions: 'promotions',
};

const backendSortableFields: SortField[] = ['name', 'credits', 'type', 'seat', 'conflict', 'fallback', 'promotions'];
```

**Why in-memory sorting is acceptable**:
- The course list is admin-only, typically hundreds to low-thousands of records
- The `no_pagination` flag already supports fetching all records (up to 10,000)
- Computed fields depend on multiple service calls and JOINs that can't be expressed as simple DQL ORDER BY
- Sorting by `fallback` (always 0) is technically a no-op but included for UI consistency

**Alternative considered**: Adding SQL subqueries for `seat` and `conflict` computations. Rejected because:
- `seat` requires counting bids by status (enrolled/invited/waitlisted) per class — complex correlated subquery
- `conflict` is stored as a comma-separated string, requiring string manipulation in SQL
- Both approaches would complicate the already-complex `searchQueryPaginated()` DQL query
- In-memory sort is simpler, safer, and sufficient for the dataset size

## Risks / Trade-offs

- **Risk: `fetchJoinCollection: false` causes incorrect pagination** → Mitigation: The `GROUP BY cl.id` already ensures one row per class, making the Paginator's subquery-based dedup unnecessary. The count is already overridden via `totalOverride`. Tested: the non-campaign mode path with `fetchJoinCollection: true` remains untouched.
- **Risk: GROUP BY changes query performance in StudentRepository** → Mitigation: GROUP BY on primary key (`s.id`) is functionally equivalent to DISTINCT on the entity and should not impact performance. MySQL optimizes `GROUP BY primary_key` efficiently.
- **Risk: Paginator behavior with GROUP BY in StudentRepository** → Mitigation: The Paginator is already instantiated with `fetchJoinCollection: false`, so the count query will work correctly with GROUP BY.
- **Risk: Changing sort field names breaks existing saved state** → Mitigation: Sort state is ephemeral (in React state), not persisted. Users will simply get a fresh default sort on next page load.
- **Risk: `cl.section` ORDER BY in campaign group mode** → Mitigation: `cl.section` is a column on the Class entity (root entity, alias `cl`), which is included in the GROUP BY clause. ORDER BY on grouped columns is always valid.
- **Risk: Adding `class_id` breaks existing API consumers** → Mitigation: Additive field only. Existing consumers ignore unknown fields. No existing field is changed or removed.
- **Risk: In-memory sorting fetches all records for computed-field sorts** → Mitigation: The course list is admin-only with typically hundreds to low-thousands of records. The existing `no_pagination` flag already supports fetching up to 10,000 records. The `maxNoPaginationLimit` (10,000) cap prevents unbounded memory usage. For normal-sized datasets this is negligible overhead.
- **Risk: In-memory sort pagination metadata differs from SQL pagination** → Mitigation: The controller explicitly rebuilds `Pagination` after slicing, so `totalRecords`, `currentPage`, and `totalPages` remain accurate.
