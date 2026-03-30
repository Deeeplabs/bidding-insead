# Design - Fix Bidding Page Ranking Sync After Navigation

## Problem Root Cause
The `bidding-web` frontend utilizes `@tanstack/react-query` for state management and data fetching. The global `QueryClient` configuration in `src/lib/api/query-client.ts` sets `staleTime: 5 * 60 * 1000` (5 minutes) and `refetchOnMount: false`. 

While these settings optimize performance by reducing redundant API calls, they cause synchronization issues for the campaign detail query (`activeCampaign`). When a student performs a save mutation (`useSubmitRankings`), the query for that campaign is invalidated. However, because `refetchOnMount` is `false`, navigating back to the bidding page does not trigger a re-fetch of the now-stale query. Instead, the user is presented with the cached (stale) data until either the 5-minute `staleTime` expires or a manual page refresh is performed.

## Implementation Details

### Web - bidding-web (Frontend)

#### `src/lib/api/queries/campaign.queries.ts`
- Modify `campaignQueries.activeCampaign` to include `refetchOnMount: true` in its `queryOptions`.
- Modify `campaignQueries.waitlistStatus` to include `refetchOnMount: true` in its `queryOptions` to prevent similar issues in the Waitlist phase.

By setting `refetchOnMount: true`, React Query will check if the query is marked as STALE whenever the component using it mounts. If it is stale (which it will be after a successful save mutation due to `invalidateQueries`), it will immediately trigger a re-background fetch.

Note: We keep `staleTime` at 5 minutes to avoid excessive fetches for other non-critical interactions, but `refetchOnMount: true` ensures that state-changing operations (which invalidate the query) are correctly reflected upon navigation.

### Impacted Files
- `bidding-web/src/lib/api/queries/campaign.queries.ts`

### Testing Strategy
1.  **Manual Verification (Chrome DevTools)**:
    -   Open Bidding page.
    -   Add courses/rank them.
    -   Click "Save". Observe the network tab: `PUT /api/student/bidding/{id}/rankings` succeeds.
    -   Observe that the `activeCampaign` query is marked as "stale" in React Query DevTools (if available) or simply check that `invalidateQueries` was called.
    -   Navigate to "Profile".
    -   Navigate back to "Bidding".
    -   Verify that a new `GET /api/student/campaign/{id}` request is fired immediately upon mount.
    -   Verify that the "Your Ranked Courses" list correctly reflects the saved data.

2.  **Regression Testing**:
    -   Verify that the `activeCampaigns` list (plural) still works correctly (it already has `refetchOnMount: true`).
    -   Verify that other screens like `BidSubmissionPage` (which also uses `activeCampaign`) correctly sync changes after navigation.

### Security and Performance
- No new security risks.
- Performance impact: Slight increase in API calls for the `activeCampaign` endpoint when navigating back to the bidding page, but only if the data was already stale or invalidated. This is the desired behavior for data consistency.
