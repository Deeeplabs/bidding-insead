## 1. Backend Fix — Array Parameter Handling in listCourse()

- [x] 1.1 **Fix `campus_ids` retrieval in `listCourse()`**
  - File: `bidding-api/src/Controller/Api/FlexSwitch/FlexSwitchController.php`
  - Method: `listCourse()`, line 134
  - Replace `$request->query->get('campus_ids')` with `$request->query->all('campus_ids')`
  - Simplify downstream handling: `all()` returns an array directly, so remove the `is_string` / `json_decode` / `explode` branching
  - Keep backward compat: if a single comma-separated string is passed (e.g., `campus_ids=1,2,3`), detect and explode it

- [x] 1.2 **Fix `modes` retrieval in `listCourse()`**
  - Same file, method `listCourse()`, line 159
  - Replace `$request->query->get('modes')` with `$request->query->all('modes')`
  - Apply same simplification pattern as `campus_ids`

## 2. Backend Fix — Array Parameter Handling in listClassConfiguration()

- [x] 2.1 **Fix `campus_ids` and `modes` retrieval in `listClassConfiguration()`**
  - Same file, method `listClassConfiguration()`, lines 41-42
  - Replace `$request->query->get('campus_ids')` with `$request->query->all('campus_ids')`
  - Replace `$request->query->get('modes')` with `$request->query->all('modes')`
  - Note: this method passes filters as a simple array to the service — ensure the array values are passed correctly

## 3. Backend Fix — Array Parameter Handling in moduleDetail()

- [x] 3.1 **Fix `campus_ids` retrieval in `moduleDetail()`**
  - Same file, method `moduleDetail()`, line 358
  - Replace `$request->query->get('campus_ids')` with `$request->query->all('campus_ids')`

## 4. Manual Verification

- [ ] 4.1 **Verify campus filter works via API**
  - Call `GET /flex-switch/list-course?page=1&limit=10&programmeId=2&campus_ids[]=2`
  - Confirm response returns `success: true` with filtered results (no 400 error)
  - Test with multiple campus IDs: `campus_ids[]=2&campus_ids[]=3`

- [ ] 4.2 **Verify mode filter works via API**
  - Call `GET /flex-switch/list-course?page=1&limit=10&modes[]=online`
  - Confirm response returns `success: true` with filtered results

- [ ] 4.3 **Verify no filter still works**
  - Call `GET /flex-switch/list-course?page=1&limit=10` (no campus/mode params)
  - Confirm response returns all items without filtering

- [ ] 4.4 **Verify in admin UI**
  - Login as PM in `bidding-admin`
  - Go to Switch → All Courses
  - Select a campus filter from the filter bar → verify results are filtered correctly
  - Clear the filter → verify all results return

- [ ] 4.5 **Regression check**
  - Verify `GET /flex-switch-class-configuration` with `campus_ids[]=2` works
  - Verify module detail endpoint with `campus_ids[]=2` works
  - Verify all three endpoints still work without any campus/mode filters
