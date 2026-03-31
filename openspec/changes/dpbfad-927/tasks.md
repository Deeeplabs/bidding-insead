## 1. Backend Implementation (bidding-api)

- [x] 1.1 Update the `/v2/api/student/flex-switch/calendar` API endpoint (and its corresponding domain service) responsible for providing the dashboard calendar module data.
- [x] 1.2 Implement the filtering logic: extract the requesting student's programme and active campus (including `Exchange` records).
- [x] 1.3 Update the module query to exclude any campaign modules that do not match the student's current programme and campus.
- [x] 1.4 Write or update PHPUnit tests in `tests/Unit/Domain/` to verify that the returned module array accurately reflects only the student's allowed context.

## 2. Verification

- [ ] 2.1 Start the local environment and log in as a student with a specific programme (e.g., GEMBA SGP).
- [ ] 2.2 Verify that the dashboard calendar view natively renders only relevant modules without any frontend code changes.
- [ ] 2.3 Verify that the Switch Request dropdown still correctly lists all available modules across campaigns.
- [x] 2.4 Stop the local development environment and run final unit tests.
