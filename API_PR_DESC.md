# Fix: Waitlist Limit Per Bidding Cycle Not Enforced

## Problem
Jira: [DPBFAD-718](https://insead.atlassian.net/browse/DPBFAD-718)

The Programme Manager can configure the maximum number of waitlisted courses a student may retain per bidding cycle via the Add/Drop & Waitlist module configuration. However, this limit was not enforced due to three interrelated bugs:

1. **Config key mismatch**: The admin UI saves the per-student waitlist limit as `waitlist_capacity`, but backend services read `max_waitlist_slots_per_student` (never set by admin) — causing enforcement to silently skip.
2. **Validator never called**: `AddDropValidator::validateMaxWaitlistSlotsPerStudent()` existed but was never wired into `AddDropService::submitAddDrop()`.
3. **Wrong module lookup for display**: `EnrollmentViewService::getMaxWaitlistAllowed()` read from the `final_enrollment` module instead of `add_drop_waitlist`, and used incorrect config keys (`max_waitlist_per_student`, `max_waitlist_allowed`). It also contained dead code (`$campaignConfig = null ?? []`).

## Changes Made

### 1. `AddDropValidator.php` — Fix config key resolution + respect toggle
- Added `limit_waitlist_capacity` toggle check — if `false` or not set, skip enforcement entirely.
- Added `waitlist_capacity` as fallback after `max_waitlist_slots_per_student` so existing configs saved by admin UI are recognized.

### 2. `AddDropService.php` — Wire validator into submission flow
- Added call to `$this->validator->validateMaxWaitlistSlotsPerStudent()` after bid points validation (Step 9), before executing the add/drop operation. This enforces the per-student waitlist limit during student submissions.

### 3. `EnrollmentViewService.php` — Fix module lookup + config keys
- Changed module lookup from `getCurrentActivePhaseDetailsByModule('final_enrollment')` to `getCurrentActivePhaseDetailsByModule('add_drop_waitlist')`.
- Replaced multi-fallback dead code with clean single-source lookup: `max_waitlist_slots_per_student` → `waitlist_capacity` → `0`.
- Added `limit_waitlist_capacity` toggle check — returns `0` (unlimited) when toggle is disabled.
- Removed dead-code block (`$campaignConfig = null ?? []`) and ineffective module iteration fallback.

### 4. `CampaignWaitlistService.php` — Fix config key fallback
- Added `waitlist_capacity` to the fallback chain in `getMaxWaitlistPerStudent()`: `max_waitlist_slots_per_student` → `waitlist_capacity` → `max_waitlist_per_student` → `0`.

### 5. Unit Tests — `AddDropValidatorWaitlistSlotsTest.php`
- Test: `waitlist_capacity` fallback enforces limit when `max_waitlist_slots_per_student` is absent.
- Test: No validation when `limit_waitlist_capacity` toggle is `false`.
- Test: `max_waitlist_slots_per_student` takes precedence over `waitlist_capacity` when both present.

## Impact
- **No Database Migrations**: All changes are behavioral fixes in PHP service logic.
- **Backward Compatible**: Existing configs storing the limit as `waitlist_capacity` continue to work via fallback. The `CampaignToModuleDetailDtoMapper` already had the correct fallback and required no changes.
- **No API contract changes**: `max_waitlist_allowed` field in response DTOs is unchanged — it now returns the correct value.

## Verification Steps
1. As PM, configure waitlist limit to 5 in Add/Drop & Waitlist module config (Enable Waitlist Capacity toggle ON).
2. As student, verify the Add/Drop page shows `Max waitlist allowed: 5`.
3. As student, attempt to retain more than 5 waitlisted courses — verify submission is blocked with error message.
4. Disable the "Enable Waitlist Capacity" toggle — verify no limit is enforced and `Max waitlist allowed` shows 0.
5. Verify existing campaigns with previously saved configs still display and enforce limits correctly.
