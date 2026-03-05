# Display Actual vs Expected Campaign Module Dates

## Problem

When a Programme Manager (PM) duplicates a campaign, or manually kicks off a phase ahead of schedule, the interface required an audit pattern to correctly show the `actual` execution bounds against the initially configured `expected` parameters. The module list blindly consumed `start_date`/`end_date` without providing the PM context around manual intervention. Furthermore, duplicated campaigns historically carried over module statuses, producing visually broken "filter" statuses that appeared active for brand new campaigns.

## Solution

Integrated frontend logic to consume decoupled `actual` vs `expected` configuration dates seamlessly, outputting a precise `Actual Start–End Date (Expected Start–End Date)` UI footprint if an active override is detected during campaign operations.

### Changes Made

**Modified Files:**

1. **`src/components/campaign/box-campaign.tsx`**
   - Updated module timeline text parser to condition the output against newly supplied `module_config?.actual_start_date` and `actual_end_date`.
   - If an `actual_*` date is present from the backend, the timeline securely outputs `Actual [...dates...] (Expected [...dates...])`.
   - Native fallback remains securely locked onto traditional rendering if no PM overrides occurred.
   1. **`src/src/campaign-management/campaign.types.ts`**
   - Added `actual_start_date`, `actual_end_date`, `expected_start_date`, `expected_end_date` to `ModuleConfig` type.

## Impact

- **Data Visibility:** Allows PM to clearly track where execution dates actively deviated from campaign deployment strategy. 
- **Consistency:** Ties directly into the isolated `fix-status-duplicate-bidding-phases` backend update safely. Duplicated campaigns correctly show fresh module configurations without inherited visual glitches.

## Testing

- [x] Verified tooltips and string rendering in the module timeline natively toggles to parenthetical layout if `actual_start_date` is fed through context.
- [x] Fallback layout untouched and formats dates traditionally.
