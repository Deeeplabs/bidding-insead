## 1. Backend Implementation (bidding-api)

- [x] 1.1 Update the `/v2/api/student/flex-switch/calendar` API endpoint's `getCalendarView()` method in `FlexSwitchService.php` to look up the FLEX campus (short_name = "FLEX") via `CampusRepository` and add its ID to the `$allowedCampusIds` array alongside the student's home campus and exchange campuses.
- [x] 1.2 Inject the `CampusRepository` dependency into `FlexSwitchService` (or use the EntityManager to query the Campus entity) to resolve the FLEX campus by `shortName`.
- [x] 1.3 Ensure the FLEX campus lookup is graceful: if no campus with short_name "FLEX" exists, skip it without error.
- [x] 1.4 Write or update PHPUnit tests in `tests/Unit/Domain/FlexSwitch/FlexSwitchCalendarFilterTest.php` to verify: (a) FLEX campus modules are included for all students, (b) filtering still works correctly for home campus and exchange campuses, (c) no error when FLEX campus does not exist.

## 2. Verification

- [ ] 2.1 Start the local environment and log in as a student with a specific programme (e.g., GEMBA SGP).
- [ ] 2.2 Verify that the dashboard calendar view renders modules for the student's home campus AND FLEX campus modules.
- [ ] 2.3 Verify that the Switch Request dropdown still correctly lists all available modules across campaigns.
- [ ] 2.4 Run final unit tests to confirm no regressions (`php bin/phpunit tests/Unit/Domain/FlexSwitch/FlexSwitchCalendarFilterTest.php`).
