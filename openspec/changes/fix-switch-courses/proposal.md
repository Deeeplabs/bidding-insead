## Why

In the PM admin panel under Switch → All Courses, filtering by campus causes a 400 Bad Request error. When the frontend sends `campus_ids[]=2` as a query parameter, the endpoint returns:

```json
{
  "success": false,
  "message": "Input value \"campus_ids\" contains a non-scalar value.",
  "error": { "code": "BAD_REQUEST" }
}
```

**Root cause**: Symfony 6.4's `InputBag::get()` throws `BadRequestHttpException` when the query parameter value is non-scalar (i.e., an array). The frontend (`bidding-admin`) sends `campus_ids` as `number[]` via axios, which serializes to `campus_ids[]=2` in the URL. PHP parses this as an array. When `FlexSwitchController::listCourse()` calls `$request->query->get('campus_ids')` (line 134), Symfony throws before the controller logic ever runs. The `ExceptionSubscriber` catches this and formats it as the error response above.

The same vulnerability exists for the `modes` parameter (line 159), and in two other controller methods: `listClassConfiguration()` (line 41) and `moduleDetail()` (line 358).

## What Changes

Replace `$request->query->get()` with `$request->query->all()` for array-type query parameters (`campus_ids`, `modes`) across all affected methods in `FlexSwitchController`. The `all()` method is Symfony's designated way to retrieve array values from the query bag without triggering the non-scalar exception.

## Capabilities

### New Capabilities
_(none)_

### Modified Capabilities
- `flex-switch-list-course`: Fix `campus_ids` and `modes` array parameter handling so campus/mode filters work correctly.
- `flex-switch-class-configuration`: Fix `campus_ids` and `modes` array parameter handling (same pattern).
- `flex-switch-module-detail`: Fix `campus_ids` array parameter handling (same pattern).

## Impact

- **API (`bidding-api`)**: `FlexSwitchController` — three methods need `get()` → `all()` for array params.
- **Affected endpoints**:
  - `GET /flex-switch/list-course` (PM → Switch → All Courses tab)
  - `GET /flex-switch-class-configuration` (PM → Switch → Class Configuration tab)
  - `GET /flex-switch/{id}/{mode}/{campus}/module-detail` (PM → Switch → Calendar module detail)
- **Affected file**:
  - `src/Controller/Api/FlexSwitch/FlexSwitchController.php` — `listCourse()`, `listClassConfiguration()`, `moduleDetail()` methods
- **No migration required**: This is a controller-level parameter retrieval fix.
- **No breaking API changes**: The response shape is unchanged; array filters will now work instead of returning 400.
- **No frontend changes needed**: The frontend already sends array parameters correctly via axios.
