# PM Credit Progress Fixes

## Problem
Currently, the student and PM dashboards do not accurately reflect the allocated credits and capital or what has been spent/fulfilled during bidding cycles when using the new PM campaign configurations. The calculations were relying on older legacy modes (e.g., specific session-based configurations) and ignoring the campaign rules set by the Program Manager for minimum credits and maximum capital. Additionally, the PM dashboard (`bidding-admin`) lacked parity with the backend `StudentCreditService`/`StudentCapitalService`, specifically missing the `credits_to_be_fulfilled` field in its DTOs, leading to discrepant progress visualization.

## Goal
Ensure parity and accuracy between the actual PM campaign configurations and the metrics displayed on both the Student and PM Dashboards, explicitly updating "Credit Earned", "Credits to be fulfilled", "Capital Spent", and "Capital Left".

## Changes Made

1. **`src/Repository/BidRepository.php`**
   - Added support for fetching total bidding credits by bounding to specific campaigns in `findTotalBiddingCreditsByStudent()` and `findTotalBiddingCreditsByStudents()`. This ensures that "Credit Earned" strictly sums the Final Enrollment Courses Credits and adjustments for the active bidding rounds designated by the PM.

2. **`src/Domain/Student/StudentCreditService.php`**
   - Updated `getStudentTotalCreditsTaken()`: It now queries for active PM campaigns via `getStudentEligibleCampaigns()`. If the student is part of an active campaign, it calculates the taken credits accurately using `BidRepository::findTotalBiddingCreditsByStudent()` tied to the active campaign IDs.
   - Updated `getTotalCredits()`: It now similarly passes active `campaignIds` to accurately reflect the correct domain bounded sum of total available credits.

3. **`src/Domain/Student/StudentCapitalService.php`**
   - Updated `getStudentTotalSpentCapital()`: It now fetches active campaigns. If campaigns exist, the spent capital exclusively relies on `findTotalPointsByStudentAndCampaigns()` to prevent "double-counting" against legacy fallback calculations (`$spentFromPeriods` and `$spentFromExchanges`).
   - Updated `getBatchCapital()`: Optimized the batching logic. While calculating initial capital, it detects if a student is under campaign mode and evaluates their spent capital utilizing `findTotalPointsByStudentAndCampaigns()` directly, skipping the legacy batch `bids` fallback array reduction to prevent redundancy and correct the metric.

4. **`src/Domain/Student/StudentDto.php` & `src/Domain/Student/StudentListDto.php`**
   - Introduced `credits_to_be_fulfilled` (`float`) property into both DTOs to guarantee real-time reflection of the configured PM goals to the `bidding-admin` frontend (PM dashboard).

5. **`src/Domain/Student/Mapper/StudentToDtoMapper.php` & `src/Domain/Student/Mapper/StudentListToDtoMapper.php`**
   - Updated logic to map the newly introduced `credits_to_be_fulfilled` property from `$this->studentCreditService->getTotalCreditGranted()`.

## Impact
- Core Domain calculations accurately interpret PM configuration for minimum credits and capital grants dynamically instead of statically.
- The `bidding-api` cleanly reflects `credits`, `credits_to_be_fulfilled`, `capital_left`, and `capital_spent` identically to PM dashboards and Student Dashboards natively.

## Testing / Verification Steps
- PM Dashboard dynamically displays accurate `credits_to_be_fulfilled` and balances matching the student stats.
- Capital spent accurately captures real-time bidding without double counting periods overlaying PM cycles.
