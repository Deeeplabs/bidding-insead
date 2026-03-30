# Fix: Ensure Bidding Page Refreshes After Navigation

## Problem
**Jira**: [DPBFAD-920](https://insead.atlassian.net/browse/DPBFAD-920)

In the student portal's pre-bidding step, after saving course rankings and navigating to other screens (like the profile), then returning to the bidding page, the saved course rankings were not visible in the "Your Ranked Courses" list. Users had to manually reload the entire page to see their saved changes. This was caused by the `activeCampaign` query having a long `staleTime` and `refetchOnMount: false`, which prevented it from re-fetching even after being invalidated by a save mutation.

## Goal
- Ensure that the bidding page always displays the most up-to-date saved rankings when a student navigates back to it.
- Maintain consistent data synchronization across all critical bidding and waitlist views.

## Changes Made (bidding-web)

### 1. `bidding-web/src/lib/api/queries/campaign.queries.ts`
- Updated `campaignQueries.activeCampaign` and `campaignQueries.waitlistStatus` to include `refetchOnMount: true`.
- This forces React Query to re-validate the data in the background whenever the component mounts if the query is marked as stale (which happens after any successful save or submission mutation).

## Impact
- **No API changes required** — Frontend-only fix utilizing existing invalidation patterns.
- **Improved UX** — Fixed the "disappearing rankings" bug; students now see their saved data immediately after navigating back to the bidding page.
- **Improved Consistency** — The same synchronization logic is applied to the individual campaign detail and waitlist status views.

## Testing / Verification Steps
1. Navigate to the Pre-Bidding page for an active campaign.
2. Add several courses and change their ranks.
3. Click **Save** and wait for the "Ranking saved!" notification.
4. Navigate to the **Profile** page or **Dashboard**.
5. Navigate back to the **Bidding** page.
6. Verify that a network request for the campaign detail is triggered and the "Your Ranked Courses" list correctly displays the previously saved ranks.
7. Repeat the same flow for **Bid Submission** and **Waitlist** pages to ensure consistent behavior.
