# Static Seat Display & PM Dashboard Seat Format Fix

## Problem
Jira: [DPBFAD-726](https://insead.atlassian.net/browse/DPBFAD-726)

The PM's admin dashboard incorrectly shows the split seat format `XX (XX)` for courses whenever `total_all_seats !== total_seats`, even when the course is not genuinely shared across multiple programme-promotions. The condition should only trigger for courses split between promotions. Additionally, the API needs to provide an explicit `is_shared` flag so the admin frontend can correctly determine when to show the split format.

## Changes

### 1. `AdminBiddingRoundCourseDto.php` — Add `is_shared` Boolean
- Added `is_shared` boolean property with `#[OA\Property]` annotation (defaults to `false`).
- Indicates whether the course is shared/split across multiple programme-promotions.
- Used by the admin frontend to decide seat column format: `XX` vs `XX (XX)`.

### 2. `AdminCampaignDetailService.php` — Compute `is_shared` from ClassPromotions
- In `buildBiddingRoundDetail()`, compute `is_shared` for each course by counting distinct promotion IDs from the class's `ClassPromotions` collection.
- `is_shared = true` when `count(distinct promotion IDs) > 1`.
- Set `$courseDto->is_shared = $isShared` on each `AdminBiddingRoundCourseDto`.

### 3. Unit Tests — `AdminBiddingRoundCourseDtoIsSharedTest.php`
- New PHPUnit test file at `tests/Unit/Domain/Dashboard/Campaign/`.
- Verifies: (a) `is_shared` defaults to `false`, (b) `is_shared = false` for single promotion, (c) `is_shared = true` for multiple promotions, (d) `is_shared = false` for duplicate entries of the same promotion, (e) `total_seats` and `total_all_seats` computed correctly.

### 4. Unit Tests — `AddDropValidatorSeatAvailabilityTest.php`
- New PHPUnit test file at `tests/Unit/Domain/Campaign/ActiveCampaign/Validator/`.
- Verifies: (a) enrollment passes when seats are available, (b) enrollment rejected with clear "is full" error when seats exhausted, (c) classes with no seat configuration are rejected, (d) over-capacity classes are rejected.

### 5. Backend Enrollment Validation — Verified (No Changes)
- `AddDropService::executeCampaignAddDrop()` already uses pessimistic locking (`countEnrolledWithLock`) and auto-assigns WAITLISTED status when a course is full.
- `AddDropValidator::validateClassAvailability()` returns clear "is full" error messages.
- Concurrent enrollment protection is in place via `SELECT ... FOR UPDATE`.

## Impact
- **No Database Migrations**: Only DTO and service logic changes.
- **No Breaking API Changes**: `is_shared` is an additive field. Existing consumers unaffected.
- **Backward Compatible**: If admin frontend is not deployed simultaneously, old behavior continues until frontend is updated.

## Verification Steps
1. View PM dashboard bidding round detail — verify non-shared courses show only `XX` in the seat column.
2. Verify shared courses (split across promotions) show `XX (XX)` format.
3. Run PHPUnit test suite (`bin/phpunit`) — verify no regressions and new tests pass.
