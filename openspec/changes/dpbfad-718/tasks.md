## 1. Fix AddDropValidator config key resolution and respect toggle

- [x] 1.1 In `bidding-api/src/Domain/Campaign/ActiveCampaign/Validator/AddDropValidator.php`, update `validateMaxWaitlistSlotsPerStudent()` (line 435) to fall back to `waitlist_capacity`: change `$moduleConfig['config']['max_waitlist_slots_per_student'] ?? null` to `$moduleConfig['config']['max_waitlist_slots_per_student'] ?? $moduleConfig['config']['waitlist_capacity'] ?? null`
- [x] 1.2 In the same method, add a check for the `limit_waitlist_capacity` toggle: if `$moduleConfig['config']['limit_waitlist_capacity']` is `false` or not set, return early (skip enforcement)

## 2. Wire validateMaxWaitlistSlotsPerStudent into AddDropService

- [x] 2.1 In `bidding-api/src/Domain/Campaign/ActiveCampaign/AddDropService.php`, add a call to `$this->validator->validateMaxWaitlistSlotsPerStudent($enrollments, $student, $moduleConfig)` after the bid points validation (after line ~193, before Step 11 `executeCampaignAddDrop`). Pass the `$moduleConfig` array from `getAddDropModuleConfig()`.
- [x] 2.2 Also pass `$waitlist` (waitlist items) to the validator if needed — the current validator signature accepts `$enrollments` which are the new courses being added. Verify the validator correctly counts potential waitlisted items from the enrollments array.

## 3. Fix EnrollmentViewService to read from correct module with correct keys

- [x] 3.1 In `bidding-api/src/Domain/Campaign/ActiveCampaign/Enrollment/EnrollmentViewService.php`, update `getMaxWaitlistAllowed()` (line 586): change the module lookup from `getCurrentActivePhaseDetailsByModule('final_enrollment')` to `getCurrentActivePhaseDetailsByModule('add_drop_waitlist')`
- [x] 3.2 Replace the config key lookups with the standard fallback chain: `$config['max_waitlist_slots_per_student'] ?? $config['waitlist_capacity'] ?? 0`
- [x] 3.3 Remove the dead-code `$campaignConfig = null ?? []` block (lines 601-608) and the module iteration fallback (lines 611-620) — replace with the clean single-source lookup
- [x] 3.4 Check if `limit_waitlist_capacity` toggle is respected: if `false`, return `0` (unlimited)

## 4. Fix CampaignWaitlistService config key fallback

- [x] 4.1 In `bidding-api/src/Domain/Campaign/ActiveCampaign/Waitlist/CampaignWaitlistService.php`, update `getMaxWaitlistPerStudent()` (line 576-583) to add `waitlist_capacity` to the fallback chain: `$activePhase['config']['max_waitlist_slots_per_student'] ?? $activePhase['config']['waitlist_capacity'] ?? $activePhase['config']['max_waitlist_per_student'] ?? 0`

## 5. Update admin UI to send canonical config key

- [x] 5.1 In `bidding-admin/src/components/preset/configuration/add-drop-configuration.tsx`, update the payload construction (around line 57) to also include `max_waitlist_slots_per_student: values.limit_waitlist_capacity ? values.waitlist_capacity : null` alongside the existing `waitlist_capacity` field
- [x] 5.2 Verify the `AddDropWaitlistConfigRequest` DTO already supports `max_waitlist_slots_per_student` (confirmed — line 23 of `AddDropWaitlistConfigRequest.php`)

## 6. Unit tests

- [x] 6.1 Add PHPUnit test for `AddDropValidator::validateMaxWaitlistSlotsPerStudent()`: verify that when config has `waitlist_capacity = 3` (without `max_waitlist_slots_per_student`), the limit is enforced at 3
- [x] 6.2 Add PHPUnit test for `AddDropValidator::validateMaxWaitlistSlotsPerStudent()`: verify that when `limit_waitlist_capacity = false`, no validation is performed regardless of `waitlist_capacity` value
- [x] 6.3 Add PHPUnit test for `AddDropValidator::validateMaxWaitlistSlotsPerStudent()`: verify that `max_waitlist_slots_per_student` takes precedence over `waitlist_capacity` when both are present
- [ ] 6.4 Add PHPUnit test for `EnrollmentViewService::getMaxWaitlistAllowed()`: verify it reads from `add_drop_waitlist` module and returns the correct value using `waitlist_capacity` fallback
- [ ] 6.5 Add PHPUnit test for `CampaignWaitlistService::getMaxWaitlistPerStudent()`: verify `waitlist_capacity` fallback works

## 7. Smoke test

- [ ] 7.1 Manual verification: As PM, configure waitlist limit to 5 in Add/Drop config. As student, verify `max_waitlist_allowed` shows 5 on the Add/Drop page. Try to retain more than 5 waitlisted courses and verify the submission is blocked.
