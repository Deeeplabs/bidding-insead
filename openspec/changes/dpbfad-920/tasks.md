# Internal Tasks - Fix Bidding Page Ranking Sync After Navigation

## Frontend Tasks (bidding-web)

### 1. Update `activeCampaign` Query Options [x]
- File: `bidding-web/src/lib/api/queries/campaign.queries.ts`
- In `campaignQueries.activeCampaign`, add `refetchOnMount: true` to the `queryOptions` object. [x]
- Verify that this forces a re-fetch of the single campaign data when any component using it mounts, provided that the query was invalidated or became stale. [x]

### 2. Update `waitlistStatus` Query Options [x]
- File: `bidding-web/src/lib/api/queries/campaign.queries.ts`
- In `campaignQueries.waitlistStatus`, add `refetchOnMount: true` to the `queryOptions` object. [x]
- This ensures consistency for waitlist-related data during navigation back to the waitlist page. [x]

### 3. Verification [x]
- Manually test the pre-bidding page flow as described in Scenario 1.
- Open Bidding page, save rankings, navigate to any other page (e.g., Profile), then navigate back.
- Verify that a network request for the campaign detail is fired and the UI is updated with the latest saved data. [x]
- Repeat the test for the Bid Submission phase (if possible) and Waitlist phase. [x]

### 4. Regression Testing [x]
- Verify that the `activeCampaigns` (plural) infinite list still works as expected and re-fetches on mount. [x]
- Verify that navigating within the same page (e.g., between tabs if any exist that don't unmount the query) does not cause excessive re-fetches if the data is not stale. [x]
- Smoke test across different modules (Pre-Bidding, Bidding Round, Simulation, Add/Drop, Waitlist) to ensure data stability. [x]

## Expected Outcomes
- Student portal properly reflects saved rankings and bids after navigation.
- Consistent data synchronization behavior across all critical campaign-related queries.
- No more "gone" courses in the ranked list after navigating away and back.
