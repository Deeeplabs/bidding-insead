## Why

The student dashboard header currently does not accurately reflect what has been granted to students or what has been spent/fulfilled during bidding cycles. This causes confusion for PMs and students who need accurate, real-time insight into bidding activity.

## What Changes

- **Fix Capital Left Calculation**: Make capital_left calculation dynamically based on the accumulation of capital from campaigns participated, plus manual adjustments, minus spent capital
- **Fix Credits to be Fulfilled Calculation**: Make credits_to_be_fulfilled calculation based on minimum credits from campaigns the student participated in, minus credits earned

The frontend (bidding-web) expects these existing fields:
- capital_spent
- capital_left
- credits_earned
- credits_to_be_fulfilled

Only backend calculation logic needs to be fixed - no DTO or frontend changes required.

## Capabilities

### New Capabilities
- `student-dashboard-stats`: Fix calculation of capital and credits based on campaign participation accumulation

### Modified Capabilities
<!-- No spec-level changes - implementation fix within existing dashboard functionality -->

## Impact

### bidding-api (Backend)
- **StudentStatsService.php**: Uses updated calculation logic
- **StudentCapitalService.php**: Modified to calculate capital from Campaign.minCapitalGranted (accumulated from participated campaigns) + Adjustment, minus spent
- **StudentCreditService.php**: Modified to calculate credits_to_be_fulfilled from Campaign.minCreditsToFulfill (accumulated from participated campaigns)
- **BidRepository**: Used to find campaigns student has participated in

### bidding-web (Student Portal)
- **No changes required** - uses existing API response fields

### Database
- No changes expected

### Migration Risk
- Low - only calculation logic changes, no schema changes
