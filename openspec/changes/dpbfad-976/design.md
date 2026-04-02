## Root Cause Analysis

When a PM sets seat capacity to "3" in the bidding configuration, this value is stored in the `adjustment_course` table (`AdjustmentCourse.seatCapacity`). The admin dashboard correctly reads from this table. However, multiple student-facing code paths bypass the adjustment table and read directly from `$class->getTotalPromotionSeats()`, which sums `ClassPromotions.promotionSeats` — the original default value (e.g., 90).

### Correct Pattern (already used in some places)
```php
// Fallback chain: simulation_adjustment > adjustment > class default
$totalSeats = ($simulationAdjustmentsMap[$class->getId()] ?? null)?->getSeatCapacity()
    ?? ($adjustmentConfigsMap[$class->getId()] ?? null)?->getSeatCapacity()
    ?? $class->getTotalPromotionSeats();
```

### Broken Pattern (causes the bug)
```php
// Reads only from ClassPromotions — ignores PM overrides
$totalSeats = $class->getTotalPromotionSeats();
```

### Affected Code Paths

| File | Line(s) | Context | Uses Adjustment? |
|------|---------|---------|-----------------|
| `CampaignToModuleDetailDtoMapper.php` | 224 | `buildBiddingRoundDetail()` — submitted bids | ❌ |
| `CampaignToModuleDetailDtoMapper.php` | 584-586, 689-691 | `buildAddDropDetail()` — enrolled/waitlisted bids | ✅ |
| `CampaignToActiveBiddingRoundDtoMapper.php` | 363, 400 | Submitted bids + ranked courses | ❌ |
| `CampaignToActiveBiddingRoundDtoMapper.php` | 612, 652, 691 | Enrolled/waitlisted bids | ❌ |
| `StudentActiveCampaignController.php` | 779 | `getAvailableCourses()` | ✅ |
| `AdminCampaignDetailService.php` | 698, 959-961 | Admin dashboard views | ✅ |
| `BidValidator.php` | 177 | Bid validation | ❌ |
| `AddDropValidator.php` | 135, 396, 457 | Add/drop validation | ❌ |
| `CampaignWaitlistService.php` | 67, 163, 474 | Waitlist capacity checks | ❌ |
| `WaitlistAutoOfferService.php` | 96 | Auto-offer seat check | ❌ |

## Proposed Design

The fix follows a consistent pattern: inject `AdjustmentCourseRepository` (and `SimulationAdjustmentCourseRepository` where appropriate), load adjustment maps once per request, and replace all bare `$class->getTotalPromotionSeats()` calls with the fallback chain.

### 1. Fix `CampaignToModuleDetailDtoMapper.php` — `buildBiddingRoundDetail()`

The mapper already has `AdjustmentCourseRepository` and `SimulationAdjustmentCourseRepository` injected (used by `buildAddDropDetail()`). Apply the same pattern to `buildBiddingRoundDetail()`:

```php
// Before (line 224):
$totalSeats = $class->getTotalPromotionSeats();

// After:
$totalSeats = ($adjustmentConfigsMap[$class->getId()] ?? null)?->getSeatCapacity()
    ?? $class->getTotalPromotionSeats();
```

Load `$adjustmentConfigsMap` at the top of the method (same as `buildAddDropDetail()` does):
```php
$adjustmentConfigsMap = $this->adjustmentCourseRepository->findByCampaignIndexedByClass($campaign);
```

### 2. Fix `CampaignToActiveBiddingRoundDtoMapper.php`

This mapper does NOT currently inject `AdjustmentCourseRepository`. Changes:

1. Add `AdjustmentCourseRepository` to the constructor
2. In each method that builds course DTOs, load the adjustment map once:
   ```php
   $adjustmentMap = $this->adjustmentCourseRepository->findByCampaignIndexedByClass($campaign);
   ```
3. Replace all occurrences:
   ```php
   // Before:
   $totalSeats = $class->getTotalPromotionSeats();
   
   // After:
   $totalSeats = ($adjustmentMap[$class->getId()] ?? null)?->getSeatCapacity()
       ?? $class->getTotalPromotionSeats();
   ```

Affected methods and lines:
- Line 363: submitted bids loop
- Line 400: ranked courses loop
- Lines 612, 652, 691: enrolled/waitlisted bid loops

### 3. Fix `BidValidator.php`

Add `AdjustmentCourseRepository` to constructor. Load adjustment map and pass it to the validation method, or accept it as a parameter:

```php
// Before (line 177):
if ($class->getTotalPromotionSeats() <= 0) {

// After:
$adjustedSeats = ($adjustmentMap[$class->getId()] ?? null)?->getSeatCapacity()
    ?? $class->getTotalPromotionSeats();
if ($adjustedSeats <= 0) {
```

### 4. Fix `AddDropValidator.php`

Same pattern — inject `AdjustmentCourseRepository`, load map once, replace bare calls:

- Line 135: `$totalSeats = $class->getTotalPromotionSeats();` → use adjustment
- Line 396: same
- Line 457: same

### 5. Fix `CampaignWaitlistService.php`

Inject `AdjustmentCourseRepository`, load map once per campaign context:

- Line 67: `$totalSeats = $class->getTotalPromotionSeats() ?? 0;` → use adjustment
- Line 163: `$capacity = $class->getTotalPromotionSeats() ?? 0;` → use adjustment
- Line 474: `$capacity = $class->getTotalPromotionSeats() ?? 0;` → use adjustment

### 6. Fix `WaitlistAutoOfferService.php`

Inject `AdjustmentCourseRepository`, load map:

- Line 96: `$totalSeats = $class->getTotalPromotionSeats();` → use adjustment

## Affected Files Summary

| File Path | Impact | Rationale |
|-----------|--------|-----------|
| `bidding-api/src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToModuleDetailDtoMapper.php` | Backend | Load adjustment map in `buildBiddingRoundDetail()` |
| `bidding-api/src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapper.php` | Backend | Inject `AdjustmentCourseRepository`, use adjustment map in all seat reads |
| `bidding-api/src/Domain/Campaign/ActiveCampaign/Validator/BidValidator.php` | Backend | Inject repo, use adjusted seats for validation |
| `bidding-api/src/Domain/Campaign/ActiveCampaign/Validator/AddDropValidator.php` | Backend | Inject repo, use adjusted seats for validation |
| `bidding-api/src/Domain/Campaign/ActiveCampaign/Waitlist/CampaignWaitlistService.php` | Backend | Inject repo, use adjusted seats for capacity |
| `bidding-api/src/Domain/Campaign/ActiveCampaign/Waitlist/WaitlistAutoOfferService.php` | Backend | Inject repo, use adjusted seats for auto-offer |

## Risk Assessment

- **No migration required**: Uses existing `AdjustmentCourse` entity and `findByCampaignIndexedByClass()` repository method.
- **Backward compatible**: When no `AdjustmentCourse` record exists, the fallback chain returns `$class->getTotalPromotionSeats()` — identical to current behavior.
- **Performance**: `findByCampaignIndexedByClass()` is a single query per campaign, already used in other code paths. Adding it to the bidding round mapper adds one query per detail request.
- **No frontend changes**: The API response shape is unchanged; only the `total_seats` and `available_seats` values will now correctly reflect the PM's configuration.
- **Existing tests**: Unit tests that mock `getTotalPromotionSeats()` should continue to pass. New tests should verify the adjustment override is applied.
