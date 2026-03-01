# Fix Student Dashboard Credits and Capital Display

## Problem

The student dashboard header was not accurately reflecting what has been granted to students or what has been spent/fulfilled during bidding cycles. The calculations for capital_left and credits_to_be_fulfilled were not based on the PM's configuration in:
- Core Configuration (max credits, max bidding capital)
- Bidding Round Configuration (pre-bidding, bidding round, module limits)

### Root Cause

1. **Capital Calculation Issue:** The capital_left calculation was using PromotionSetting for initial capital, but should have been using the accumulation of minCapitalGranted from all campaigns the student participated in.
2. **Credits Calculation Issue:** The credits_to_be_fulfilled was using minCapitalGranted (capital) instead of minCreditsToFulfill (credits) from campaigns.
3. **Campaign Participation:** The system was not correctly identifying which campaigns a student participated in to calculate the accumulated values.

## Solution

Updated the backend calculation logic to dynamically reflect student allocations according to the active bidding cycle configuration:

### Changes Made

**Modified Files:**

1. **`src/Domain/Student/StudentCapitalService.php`**
   - Added BidRepository to find campaigns the student participated in
   - Modified `getInitialCapital()` to use `Campaign.minCapitalGranted` accumulated from all campaigns where student has placed bids
   - Added `getStudentParticipatedCampaigns()` method to find distinct campaigns via Bid entity

2. **`src/Domain/Student/StudentCreditService.php`**
   - Modified `getTotalCreditGranted()` to use `Campaign.minCreditsToFulfill` from participated campaigns
   - Updated to find campaigns via Bid entity instead of session periods

3. **`src/Domain/Student/Dashboard/StudentStatsService.php`**
   - Fixed credits_to_be_fulfilled calculation to subtract credits_earned from credits granted
   - Added proper calculation: credits_to_be_fulfilled = max(0, credits_granted - credits_earned)

## Files Changed

| File | Change |
|------|--------|
| `src/Domain/Student/StudentCapitalService.php` | Calculate capital from Campaign.minCapitalGranted via Bid entity |
| `src/Domain/Student/StudentCreditService.php` | Calculate credits from Campaign.minCreditsToFulfill via Bid entity |
| `src/Domain/Student/Dashboard/StudentStatsService.php` | Fix credits_to_be_fulfilled calculation |

## Impact

- **Data Accuracy:** Dashboard now accurately displays capital_left based on accumulation from campaigns participated + manual adjustments - spent
- **Data Accuracy:** Credits to be fulfilled now correctly shows minimum credits from campaigns - credits earned
- **Compatibility:** No frontend changes required - existing API response fields maintained (capital_spent, capital_left, credits_earned, credits_to_be_fulfilled)
- **Performance:** Minimal impact - uses existing BidRepository queries with deduplication

## Testing

Verified through code review:
- [x] Capital calculation uses Campaign.minCapitalGranted from participated campaigns
- [x] Capital includes manual adjustments via AdjustmentRepository
- [x] Capital left = (campaign capital + adjustments) - spent
- [x] Credits to be fulfilled uses Campaign.minCreditsToFulfill
- [x] Credits to be fulfilled = max(0, credits_granted - credits_earned)
- [x] Backend-only changes - no frontend modifications needed
- [x] Backward compatible with existing API consumers
