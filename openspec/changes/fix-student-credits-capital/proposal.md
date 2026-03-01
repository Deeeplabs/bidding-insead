## Why

The student dashboard and bidding round views can currently display **negative values** for capital_left and credits. This occurs because:

1. **Root Cause (Capital)**: `StudentCapitalService::getCapital()` computes `left = capital - spent` without clamping to zero. If a student spends more capital than granted (e.g., across multiple campaigns or via manual adjustments), the result goes negative.
2. **Root Cause (Credits)**: `StudentCreditService::getCredits()` has a hardcoded `left: 1` placeholder instead of a proper calculation, and the `toBeFulfilled` field internally already uses `max(0, ...)` but the model's `left` value is incorrect.
3. **Root Cause (Bidding Round View)**: `CampaignToActiveBiddingRoundDtoMapper` computes `capital_left = capitalGranted - cumulativePoints` without clamping, allowing negative capital display in active bidding round views.

This causes confusion for PMs and students who see negative capital or incorrect credit values on their dashboards.

## What Changes

- **Fix Capital Left Calculation**: Add `max(0, ...)` guard in `StudentCapitalService::getCapital()` so `capital_left` is never negative
- **Fix Capital Left in Bidding Round View**: Add `max(0, ...)` guard in `CampaignToActiveBiddingRoundDtoMapper::buildBiddingRoundModuleData()` for the per-module capital_left calculation
- **Fix Credits Left Placeholder**: Replace the hardcoded `left: 1` in `StudentCreditService::getCredits()` with a proper calculation
- **Fix Credits To Be Fulfilled**: Update calculation in `StudentStatsService` to properly subtract credits_earned from credits granted and clamp to zero using `max(0, ...)`
- **Match PM List With Dashboard**: Update `StudentListToDtoMapper`, `StudentToDtoMapper`, and `StudentDataToDtoMapper` to dynamically compute and map `credits_to_be_fulfilled` and `capital_left`, and refactor `StudentService::listStudents` to apply computed sorting and filtering for `capital_left` instead of stale DB values.

The frontend (bidding-web) expects these existing fields:
- capital_spent
- capital_left
- credits_earned
- credits_to_be_fulfilled

Only backend calculation logic needs to be fixed - no DTO or frontend changes required.

## Capabilities

### New Capabilities
- `student-dashboard-stats`: Fix calculation of capital and credits to prevent negative values, using campaign participation accumulation

### Modified Capabilities
<!-- No spec-level changes - implementation fix within existing dashboard functionality -->

## Impact

### bidding-api (Backend)
- **StudentCapitalService.php** (line 44): Add `max(0, ...)` to `left` calculation in `getCapital()`
- **StudentCreditService.php** (line 60): Replace hardcoded `left: 1` with proper calculation
- **CampaignToActiveBiddingRoundDtoMapper.php** (line 402): Add `max(0, ...)` to `capital_left` in bidding round module
- **StudentStatsService.php**: Fixed `credits_to_be_fulfilled` calculation to subtract `credits_earned` from credits granted, wrapped in `max(0, ...)`
- **StudentListToDtoMapper.php** / **StudentToDtoMapper.php**: Replaced legacy `credits` logic with `credits_to_be_fulfilled` computation.
- **StudentDataToDtoMapper.php**: Injected `StudentCapitalService` to map dynamically computed `capital_left` into `remaining_capital` instead of using stale static DB values.
- **StudentService.php**: Refactored `listStudents` pagination querying to dynamically sort and filter the `capital_left` rather than rely on the obsolete `COALESCE(sd.remainingCapital, 0)` SQL execution.
- **BidRepository**: Used to find campaigns student has participated in (already in place)

### bidding-web (Student Portal)
- **No changes required** - uses existing API response fields

### Database
- No changes expected

### Migration Risk
- Low - only calculation logic changes, no schema changes
- No breaking changes to API response shape (fields remain the same, values become non-negative)
