# Fix Switch Courses — Array Query Parameter Handling

## Problem

In the PM admin panel under **Switch → All Courses**, filtering by campus returns a 400 Bad Request error:

```json
{
  "success": false,
  "message": "Input value \"campus_ids\" contains a non-scalar value.",
  "error": { "code": "BAD_REQUEST" }
}
```

**Root cause:** Symfony 6.4's `InputBag::get()` throws `BadRequestHttpException` when the query parameter value is non-scalar (i.e., an array). The frontend sends `campus_ids` as `number[]` via axios, which serializes to `campus_ids[]=2` in the URL. PHP parses this as an array, and `$request->query->get('campus_ids')` throws before the controller logic runs. The same issue affects the `modes` parameter.

## Solution

Replaced `$request->query->get()` with `$request->query->all()` for array-type query parameters (`campus_ids`, `modes`) across all affected controller methods. `InputBag::all()` is Symfony's designated method for retrieving array values without triggering the non-scalar exception.

### Changes Made

**Modified File:**

**`src/Controller/Api/FlexSwitch/FlexSwitchController.php`**

1. **`listCourse()`** — `campus_ids` and `modes` parameters
   - Changed `$request->query->get('campus_ids')` → `$request->query->all('campus_ids')`
   - Changed `$request->query->get('modes')` → `$request->query->all('modes')`
   - Simplified downstream handling since `all()` returns an array directly
   - Added backward compat fallback for comma-separated string format

2. **`listClassConfiguration()`** — `campus_ids` and `modes` parameters
   - Changed `$request->query->get('campus_ids')` → `$request->query->all('campus_ids') ?: null`
   - Changed `$request->query->get('modes')` → `$request->query->all('modes') ?: null`

3. **`moduleDetail()`** — `campus_ids` parameter
   - Changed `$request->query->get('campus_ids')` → `$request->query->all('campus_ids') ?: null`

## Impact

- **Affected endpoints:**
  - `GET /flex-switch/list-course` (PM → Switch → All Courses)
  - `GET /flex-switch-class-configuration` (PM → Switch → Class Configuration)
  - `GET /flex-switch/{id}/{mode}/{campus}/module-detail` (PM → Switch → Calendar module detail)
- **No breaking changes:** API response shape is unchanged; array filters now work instead of returning 400
- **No migration required:** Controller-level parameter retrieval fix only
- **Backward compatible:** Comma-separated string format still supported as fallback
- **No frontend changes needed:** The frontend already sends array parameters correctly

## Testing

- [ ] Verified `GET /flex-switch/list-course?page=1&limit=10&programmeId=2&campus_ids[]=2` returns success (no 400 error)
- [ ] Verified multiple campus filter: `campus_ids[]=2&campus_ids[]=3`
- [ ] Verified mode filter: `modes[]=online`
- [ ] Verified combined campus + mode filters work together
- [ ] Verified no-filter requests still work correctly
- [ ] Verified class configuration endpoint with `campus_ids[]=2` works
- [ ] Verified module detail endpoint with `campus_ids[]=2` works
- [ ] Verified campus filter works correctly in admin UI (Switch → All Courses → filter bar)
