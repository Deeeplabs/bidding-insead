# BP Dashboard — Promotion Card Data Source Fix

## Problem
The "Promotion" MetricCard in the Programme Governance Overview section displays `programme_governance_overview.programme_operations` from the API response. This field actually returns the count of Tier 2 Programme Operators — not the number of promotions. As a result the dashboard shows **1** while there are **5+** active promotions for the selected programme.

## Goal
Update the "Promotion" card to read the new `promotions` field from the API response, which returns the actual count of active Promotion entities (added in the companion `bidding-api` PR).

## Changes Made

1. **`src/components/dashboard/dashboard-business-partner.tsx`**
   - Changed the "Promotion" MetricCard value from `bpStats?.programme_governance_overview?.programme_operations` to `bpStats?.programme_governance_overview?.promotions`.

2. **`src/src/dashboard/stats.types.ts`**
   - Added `promotions: number` to the `programme_governance_overview` type in `BPStatsResponse`.

## Depends On
- **bidding-api PR**: Adds the `promotions` field to `GET /dashboard/bp/stats` response. This admin PR must be deployed together with (or after) the API PR.

## Testing / Verification Steps
1. Open the BP Dashboard and select a programme.
2. Verify the "Promotion" card shows the correct count of active promotions for that programme.
3. Compare against the promotion list visible in Create Campaign → Select Promotion dropdown — numbers should match.
4. Verify "Total Users" equals the sum of Business Partners + Programme Managers + Programme Operators + Students.
5. Verify "Students" matches the count shown on the Students tab.
