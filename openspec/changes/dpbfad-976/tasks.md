## Implementation Tasks

- [x] **Task 1: Fix `CampaignToModuleDetailDtoMapper::buildBiddingRoundDetail()` — Primary Bug Fix**
  - File: `bidding-api/src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToModuleDetailDtoMapper.php`
  - The mapper already has `AdjustmentCourseRepository` injected (used by `buildAddDropDetail()`).
  - At the top of `buildBiddingRoundDetail()` (~line 157), add:
    ```php
    $adjustmentConfigsMap = $this->adjustmentCourseRepository->findByCampaignIndexedByClass($campaign);
    ```
  - Replace line 224:
    ```php
    // Before:
    $totalSeats = $class->getTotalPromotionSeats();
    // After:
    $totalSeats = ($adjustmentConfigsMap[$class->getId()] ?? null)?->getSeatCapacity()
        ?? $class->getTotalPromotionSeats();
    ```
  - This is the core fix for the reported bug (seat capacity reverting to default after bid submission).

- [x] **Task 2: Fix `CampaignToActiveBiddingRoundDtoMapper` — Seat Capacity in List View**
  - File: `bidding-api/src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapper.php`
  - Inject `AdjustmentCourseRepository` into the constructor.
  - In each method that builds course DTOs for a campaign, load adjustment map once:
    ```php
    $adjustmentMap = $this->adjustmentCourseRepository->findByCampaignIndexedByClass($campaign);
    ```
  - Replace all bare `$class->getTotalPromotionSeats()` calls (lines ~363, ~400, ~612, ~652, ~691) with:
    ```php
    $totalSeats = ($adjustmentMap[$class->getId()] ?? null)?->getSeatCapacity()
        ?? $class->getTotalPromotionSeats();
    ```
  - Verify Symfony autowiring resolves the new dependency (autowire: true in services.yaml).

- [x] **Task 3: Fix `BidValidator` — Validation Uses Adjusted Seats**
  - File: `bidding-api/src/Domain/Campaign/ActiveCampaign/Validator/BidValidator.php`
  - Inject `AdjustmentCourseRepository` into the constructor.
  - In the validation method, accept or load the adjustment map for the campaign.
  - Replace line ~177:
    ```php
    // Before:
    if ($class->getTotalPromotionSeats() <= 0) {
    // After:
    $adjustedSeats = ($adjustmentMap[$class->getId()] ?? null)?->getSeatCapacity()
        ?? $class->getTotalPromotionSeats();
    if ($adjustedSeats <= 0) {
    ```

- [x] **Task 4: Fix `AddDropValidator` — Validation Uses Adjusted Seats**
  - File: `bidding-api/src/Domain/Campaign/ActiveCampaign/Validator/AddDropValidator.php`
  - Inject `AdjustmentCourseRepository` into the constructor.
  - Load adjustment map once at the start of validation.
  - Replace all bare `$class->getTotalPromotionSeats()` calls (lines ~135, ~396, ~457) with the fallback chain pattern.

- [x] **Task 5: Fix `CampaignWaitlistService` — Waitlist Uses Adjusted Seats**
  - File: `bidding-api/src/Domain/Campaign/ActiveCampaign/Waitlist/CampaignWaitlistService.php`
  - Inject `AdjustmentCourseRepository` into the constructor.
  - Load adjustment map once per method that processes a campaign.
  - Replace all bare `$class->getTotalPromotionSeats()` calls (lines ~67, ~163, ~474) with:
    ```php
    $totalSeats = ($adjustmentMap[$class->getId()] ?? null)?->getSeatCapacity()
        ?? ($class->getTotalPromotionSeats() ?? 0);
    ```

- [x] **Task 6: Fix `WaitlistAutoOfferService` — Auto-Offer Uses Adjusted Seats**
  - File: `bidding-api/src/Domain/Campaign/ActiveCampaign/Waitlist/WaitlistAutoOfferService.php`
  - Inject `AdjustmentCourseRepository` into the constructor.
  - Load adjustment map and replace line ~96:
    ```php
    // Before:
    $totalSeats = $class->getTotalPromotionSeats();
    // After:
    $totalSeats = ($adjustmentMap[$class->getId()] ?? null)?->getSeatCapacity()
        ?? $class->getTotalPromotionSeats();
    ```

- [ ] **Task 7: Unit Tests**
  - Add/update PHPUnit tests in `tests/Unit/Domain/Campaign/ActiveCampaign/Mapper/` to verify:
    - `buildBiddingRoundDetail()` returns `AdjustmentCourse.seatCapacity` when present.
    - `buildBiddingRoundDetail()` falls back to `Class.getTotalPromotionSeats()` when no adjustment exists.
  - Follow existing test patterns (arrange-act-assert, mock repositories and services).
  - Verify existing tests still pass.

- [ ] **Task 8: Smoke Test — End-to-End Verification**
  - Create a campaign with pre-bidding and bidding modules.
  - Set seat capacity to "3" for all courses in the bidding configuration.
  - Close pre-bidding, open bidding.
  - Impersonate a student, submit a bid.
  - Verify: bidding round detail page shows `total_seats = 3` (not 90).
  - Verify: PM dashboard still shows `total_seats = 3`.
  - Verify: available courses list shows `total_seats = 3`.
  - Verify: with 3 enrolled students, a 4th student's bid is correctly rejected or waitlisted.
