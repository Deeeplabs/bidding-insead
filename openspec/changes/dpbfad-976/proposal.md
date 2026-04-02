## Why

When a PM sets the seat capacity of all courses to "3" in the bidding configuration (via `AdjustmentCourse` records), the student-facing bidding round detail page shows the correct value initially. However, after a student submits a bid, the seat capacity displayed reverts to the default value "90" (the original `ClassPromotions.promotionSeats` sum).

The PM side continues to show "3" because the admin dashboard (`AdminCampaignDetailService`) correctly reads from the `AdjustmentCourse` table. The student-facing code paths, however, inconsistently apply this override.

**Root cause**: Multiple student-facing mappers and validators read seat capacity directly from `$class->getTotalPromotionSeats()` (which sums `ClassPromotions.promotionSeats` — the master data default) instead of first checking the `AdjustmentCourse` table for PM-configured overrides. The pattern `$adjustment?->getSeatCapacity() ?? $class->getTotalPromotionSeats()` is correctly used in some code paths but missing in others.

**Code paths that correctly use adjustment overrides:**
- `StudentActiveCampaignController::getAvailableCourses()` (line 779) — available courses list before bidding
- `CampaignToModuleDetailDtoMapper::buildAddDropDetail()` (lines 584-586) — add/drop detail view
- `AdminCampaignDetailService` — all admin views

**Code paths missing adjustment overrides (the bug):**
- `CampaignToModuleDetailDtoMapper::buildBiddingRoundDetail()` (line 224) — bidding round detail after bid submission
- `CampaignToActiveBiddingRoundDtoMapper` (lines 363, 400, 612, 652, 691) — active bidding round list/detail
- `BidValidator` (line 177) — bid validation checks
- `AddDropValidator` (lines 135, 396, 457) — add/drop validation checks
- `CampaignWaitlistService` (lines 67, 163, 474) — waitlist capacity checks
- `WaitlistAutoOfferService` (line 96) — auto-offer seat checks

## What Changes

- **Fix `CampaignToModuleDetailDtoMapper.php` (Backend)**: Load `AdjustmentCourse` maps in `buildBiddingRoundDetail()` and use the `$adjustment?->getSeatCapacity() ?? $class->getTotalPromotionSeats()` pattern consistently, matching how `buildAddDropDetail()` already works.
- **Fix `CampaignToActiveBiddingRoundDtoMapper.php` (Backend)**: Inject `AdjustmentCourseRepository`, load adjustment maps, and apply the same override pattern in all seat capacity reads.
- **Fix `BidValidator.php` (Backend)**: Accept adjustment maps and use adjusted seat capacity for validation.
- **Fix `AddDropValidator.php` (Backend)**: Accept adjustment maps and use adjusted seat capacity for validation.
- **Fix `CampaignWaitlistService.php` (Backend)**: Load adjustment maps and use adjusted seat capacity for waitlist checks.
- **Fix `WaitlistAutoOfferService.php` (Backend)**: Load adjustment maps and use adjusted seat capacity for auto-offer checks.

## Capabilities

### Modified Capabilities

- **Consistent Seat Capacity Display**: Student-facing bidding round detail, active bidding rounds list, and all related views display the PM-configured seat capacity from `AdjustmentCourse` instead of the default `ClassPromotions.promotionSeats` value.
- **Accurate Bid Validation**: Bid and add/drop validators use the PM-configured seat capacity when checking whether a class is full.
- **Correct Waitlist Behavior**: Waitlist services use the PM-configured seat capacity when determining available seats and auto-offer eligibility.

## Impact

- **bidding-api**: `src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToModuleDetailDtoMapper.php` — Load adjustment maps in `buildBiddingRoundDetail()`, use adjusted seat capacity.
- **bidding-api**: `src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapper.php` — Inject `AdjustmentCourseRepository`, load adjustment maps, use adjusted seat capacity in all methods.
- **bidding-api**: `src/Domain/Campaign/ActiveCampaign/Validator/BidValidator.php` — Use adjusted seat capacity for full-class checks.
- **bidding-api**: `src/Domain/Campaign/ActiveCampaign/Validator/AddDropValidator.php` — Use adjusted seat capacity for full-class checks.
- **bidding-api**: `src/Domain/Campaign/ActiveCampaign/Waitlist/CampaignWaitlistService.php` — Use adjusted seat capacity for waitlist capacity.
- **bidding-api**: `src/Domain/Campaign/ActiveCampaign/Waitlist/WaitlistAutoOfferService.php` — Use adjusted seat capacity for auto-offer eligibility.
- **Affected roles**: Students (see correct seat capacity), Programme Managers (no change — already correct).
- **No migration required**: No database schema changes. Uses existing `AdjustmentCourse` entity and repository.
- **Regression risk**: Low — changes only add an additional lookup layer that falls back to the existing behavior when no adjustment exists.
