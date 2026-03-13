## 1. Domain Service Updates

- [x] 1.1 Locate calculation functions in `bidding-api/src/Domain/Student/StudentCreditService.php` (`getStudentTotalCreditsTaken` & `getTotalCreditGranted`).
- [x] 1.2 Locate capital calculations in `bidding-api/src/Domain/Student/StudentCapitalService.php` (`getCapital`).
- [x] 1.3 Update the "Credit Earned" calculation to accurately sum the Student Final Enrollment Courses Credits for 2 bidding rounds.
- [x] 1.4 Update the "Credits to be fulfilled" calculation to sum the Minimum credits required from the PM Campaign configuration.
- [x] 1.5 Update "Capital Spent" and "Capital Left" calculations to reflect operations limited consistently to 2 bidding rounds per PM's limit logic.

## 2. API / Mapping Updates
- [x] 2.1 Navigate to `bidding-api/src/Domain/Student/Mapper/StudentToDtoMapper.php` and `StudentListToDtoMapper.php`.
- [x] 2.2 Validate and correctly map the new outputs from Domain Services onto `StudentDto` (`credits`, `capital_left`, `capital_spent`), and if necessary, introduce `credits_to_be_fulfilled` so the PM dashboard displays correctly.

## 3. Testing

- [ ] 3.1 Identify and update PHPUnit tests in `bidding-api/tests/Unit/Domain/Student/` to assert the new calculation logic for Credits and Capital based on PM configurations.
- [ ] 3.2 Verify locally that endpoints like `/api/student/dashboard` AND `/api/admin/student/{id}` return the correct field properties without schema regressions.

## 4. Integration & Smoke Testing

- [ ] 4.1 Restart the local `bidding-api` development server.
- [ ] 4.2 Verify Student Portal (`bidding-web`) shows the correct progression.
- [ ] 4.3 Verify PM Dashboard (`bidding-admin` side) accurately displays the same progress metrics via the Admin profile overview.
