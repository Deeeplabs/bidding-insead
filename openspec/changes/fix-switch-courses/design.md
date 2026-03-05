## Context

The PM admin panel's Switch → All Courses tab uses `GET /flex-switch/list-course` to display a paginated, filterable list of classes. The request flows through:

1. **Controller**: `FlexSwitchController::listCourse()` → reads query params, builds `FilterGroup`/`FilterItem` objects, calls service
2. **Service**: `FlexSwitchService::listCourse()` → calls `ClassService::getList()` with a `PaginationList`
3. **Repository**: `ClassesRepository::searchQueryPaginated()` → builds a DQL query with JOINs and pagination

**The bug**: When the frontend sends array query parameters (e.g., `campus_ids[]=2`), PHP parses them into arrays. In Symfony 6.4, `InputBag::get()` throws `BadRequestHttpException` with the message _"Input value \"campus_ids\" contains a non-scalar value."_ when attempting to retrieve an array value. This exception is thrown at the framework level — before any controller logic executes.

The controller code at line 134 (`$campusIds = $request->query->get('campus_ids')`) already has correct downstream handling for arrays, but it never reaches that logic because Symfony rejects the call.

**Affected methods** (all in `FlexSwitchController`):
- `listCourse()` — `campus_ids` (line 134), `modes` (line 159)
- `listClassConfiguration()` — `campus_ids` (line 41), `modes` (line 42)
- `moduleDetail()` — `campus_ids` (line 358)

**Frontend**: `ListCourseFlex` component sends `campus_ids` as `number[]` and `mode` as `string[]` via axios, which serializes them as `campus_ids[]=2&mode[]=online`. This is standard PHP array notation and is the correct client behavior.

**Exception handling**: `ExceptionSubscriber` catches the `BadRequestHttpException` (which implements `HttpExceptionInterface` with status 400) and formats it using the standard `ApiResponse` / `ApiError` envelope with code `BAD_REQUEST`.

## Goals / Non-Goals

**Goals:**
- Fix array parameter retrieval in all affected `FlexSwitchController` methods so campus and mode filters work
- Ensure backward compatibility with both array notation (`campus_ids[]=2`) and JSON/comma-separated string formats that the existing code already handles
- Apply the fix consistently to all three affected methods

**Non-Goals:**
- Changing the API response shape
- Changing the frontend parameter serialization
- Modifying the `ExceptionSubscriber` error handling
- Fixing unrelated issues (pagination count, etc.)

## Decisions

### Decision 1: Use `$request->query->all()` for array parameters

**Rationale**: Symfony's `InputBag::all(string $key)` is the designated method for retrieving array values from query parameters. Unlike `get()`, it does not throw on non-scalar values. When the key is absent, it returns an empty array `[]`.

**Approach**: In each affected method, replace:
```php
$campusIds = $request->query->get('campus_ids');
```
with:
```php
$campusIds = $request->query->all('campus_ids');
```

Then simplify the downstream handling: `all()` always returns an array, so the `is_string` / `json_decode` / `explode` branches are no longer needed for the array-notation case. However, to maintain backward compatibility with comma-separated string format (e.g., `campus_ids=1,2,3`), we keep a fallback. Since `all()` returns `['1,2,3']` (single-element array) for scalar input, we check if the first element contains commas and explode if needed.

### Decision 2: Simplify the array parameter handling pattern

**Rationale**: The current code has complex branching (`is_string` → `json_decode` → `explode`, `is_array` check). With `all()` always returning an array, we can simplify to a single clean pattern.

**Approach**: For `campus_ids` in `listCourse()`:
```php
$campusIds = $request->query->all('campus_ids');
if (!empty($campusIds)) {
    // Handle case where a single comma-separated string was passed
    if (count($campusIds) === 1 && is_string($campusIds[0]) && str_contains($campusIds[0], ',')) {
        $campusIds = explode(',', $campusIds[0]);
    }
    // ... build FilterItem objects as before
}
```

Same pattern for `modes`.

### Decision 3: Apply consistently to `listClassConfiguration()` and `moduleDetail()`

**Rationale**: These methods use the same `$request->query->get('campus_ids')` pattern and will hit the same error when array params are sent.

**Approach**: Apply the same `get()` → `all()` fix in both methods.

### Decision 4: File locations

- `src/Controller/Api/FlexSwitch/FlexSwitchController.php` — the only file that needs changes (three methods)

## Risks / Trade-offs

- **Risk**: `all()` returns `[]` instead of `null` when the key is absent. **Mitigation**: We use `!empty($campusIds)` which handles both `null` and `[]` correctly. The downstream filter-building code already checks for non-empty arrays.
- **Risk**: Backward compatibility with string-format params (e.g., `campus_ids=1,2,3`). **Mitigation**: We add a fallback that explodes comma-separated strings from the single-element array that `all()` returns for scalar values.
- **Low risk**: This is a one-file change in the controller layer only — no service, repository, or entity changes needed.
