# Feature: Display Student Campaigns with Latest on Top

## Problem
Jira: https://insead.atlassian.net/browse/DPBFAD-577

Students see all their campaigns (active and historical) in the bidding view, but campaigns are rendered in API response order without client-side sorting or grouping. When campaigns are loaded across multiple paginated pages, ordering may be inconsistent. There is also no visual distinction between active and completed campaigns.

## Goal
- Add client-side sorting as defense-in-depth for consistent ordering across paginated pages.
- Visually separate active campaigns from completed campaigns in the student bidding view.

## Changes Made

### 1. `bidding-web/src/features/bidding/utils/phase.util.ts`
- Updated `filterCampaignsVisibleRounds()` to sort the campaigns array itself (not just rounds within campaigns) by `start_date` descending before mapping rounds.
- Ensures correct ordering even across infinite-scroll paginated pages.

### 2. `bidding-web/src/features/bidding/components/bidding-list/ActiveRoundCard.tsx`
- Added campaign grouping: campaigns are partitioned into "Active Campaigns" (ongoing/upcoming phases) and "Completed Campaigns" (all phases ended).
- Active campaigns render above completed campaigns, each with a section header.
- Headers are omitted when only one group has campaigns to avoid visual noise.
- Imported `getPhaseStatus` from `phase.util.ts` to determine campaign lifecycle state.

## Impact
- **No API changes required** — Frontend-only changes consuming the existing response shape.
- **Improved UX** — Students see the most recent campaigns first, with clear active vs. completed grouping.

## Testing / Verification Steps
1. Open the student bidding view and verify campaigns appear with the most recent campaign at the top.
2. Verify active campaigns appear above completed campaigns with section headers when both groups exist.
3. Verify section headers are omitted when only active or only completed campaigns are present.
4. Scroll down to load more campaigns and verify sort order is maintained across paginated pages.
5. Verify polling refresh updates the order automatically without a page reload.
