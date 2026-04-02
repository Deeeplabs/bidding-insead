# Fix: Seat Capacity Resets to Default After Student Bid Submission

## Problem
Jira: [DPBFAD-976](https://insead.atlassian.net/browse/DPBFAD-976)

When a PM configures seat capacity to **3** for courses via `AdjustmentCourse` records, the student-facing bidding round detail initially displays the correct value. However, after a student submits a bid, the seat capacity reverts to the default **90** (the sum of `ClassPromotions.promotionSeats`).

The PM dashboard continues to show "3" because `AdminCampaignDetailService` correctly reads from the `AdjustmentCourse` table. The student-facing code paths inconsistently apply this override.

**Root cause**: Multiple student-facing mappers, validators, and services read seat capacity directly from `$class->getTotalPromotionSeats()` instead of first checking the `AdjustmentCourse` table for PM-configured overrides. The pattern `$adjustment?->getSeatCapacity() ?? $class->getTotalPromotionSeats()` was correctly used in some code paths (e.g., `buildAddDropDetail()`, `getAvailableCourses()`) but missing in many others.

## Changes

### 1. `CampaignToModuleDetailDtoMapper.php` — Primary Bug Fix
- Loaded `$adjustmentConfigsMap` via `findByCampaignIndexedByClass()` at the top of `buildBiddingRoundDetail()` and `buildFinalEnrollmentDetail()`.
- Replaced all bare `$class->getTotalPromotionSeats()` calls with the adjustment fallback pattern in both methods (enrolled, waitlisted, rejected sections).

### 2. `CampaignToActiveBiddingRoundDtoMapper.php` — List View Fix
- Injected `AdjustmentCourseRepository` into the constructor.
- Loaded `$adjustmentMap` in `buildBiddingRoundModuleData()` and `buildAddDropModuleData()`.
- Replaced all 5 bare `getTotalPromotionSeats()` calls across submitted bids, available courses, enrolled, allocated, and waitlisted sections.

### 3. `BidValidator.php` — Validation Fix
- Injected `AdjustmentCourseRepository` into the constructor.
- Loaded adjustment map in `validateClassStatus()`.
- Used adjusted seat capacity for the "has no available seats" check.

### 4. `AddDropValidator.php` — Validation Fix
- Injected `AdjustmentCourseRepository` into the constructor.
- Added `loadAdjustmentMap(Campaign)` method with per-campaign caching.
- Fixed 3 bare calls in `validateWaitlistCapacity()`, `validateClassAvailability()`, and `validateMaxWaitlistSlotsPerStudent()`.

### 5. `AddDropService.php` — Wire Adjustment Map
- Added `$this->validator->loadAdjustmentMap($campaign)` call before validator invocations in `submitAddDrop()`.

### 6. `CampaignWaitlistService.php` — Waitlist Fix
- Injected `AdjustmentCourseRepository` into the constructor.
- Loaded adjustment map in `getStudentWaitlistedCourses()`, `getAvailableWaitlistCourses()`, and `validateWaitlistEligibility()`.
- Fixed 3 bare `getTotalPromotionSeats()` calls.

### 7. `WaitlistAutoOfferService.php` — Auto-Offer Fix
- Injected `AdjustmentCourseRepository` into the constructor.
- Loaded adjustment map in `processAutoOffer()` and used adjusted seats for available seat calculation.

### 8. `EnrollmentViewService.php` — Enrollment View Fix
- Injected `AdjustmentCourseRepository` into the constructor.
- Loaded adjustment map and passed to `buildEnrolledCoursesData()`, `buildWaitlistedCoursesData()`, and `getOriginalBidsByType()`.
- Fixed 3 bare `getTotalPromotionSeats()` calls.

### 9. `EnrollmentDashboardService.php` — Dashboard Fix
- Used the already-loaded `$classAdjustment` to apply seat capacity override (1 bare call fixed).

## Impact
- **No Database Migrations**: Uses existing `AdjustmentCourse` entity and `AdjustmentCourseRepository::findByCampaignIndexedByClass()`.
- **Affected roles**: Students (now see correct PM-configured seat capacity). Programme Managers (no change — already correct).
- **Regression risk**: Low — each change adds an adjustment lookup that falls back to the existing `getTotalPromotionSeats()` when no `AdjustmentCourse` override exists.
- **9 files modified**, all within `src/Domain/Campaign/ActiveCampaign/`.

## Verification Steps
1. Create a campaign with bidding module. Set seat capacity to "3" for all courses in bidding configuration.
2. Close pre-bidding, open bidding. Impersonate a student, submit a bid.
3. Verify: bidding round detail page shows `total_seats = 3` (not 90).
4. Verify: PM dashboard still shows `total_seats = 3`.
5. Verify: available courses list shows `total_seats = 3`.
6. Verify: with 3 enrolled students, a 4th student's bid is correctly rejected or waitlisted.
7. Verify: waitlist and add/drop views also show `total_seats = 3`.
8. Verify: enrollment view shows `total_seats = 3`.
