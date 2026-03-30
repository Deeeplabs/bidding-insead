# Spec - Fix Bidding Page Ranking Sync After Navigation

## Problem Description
In the student portal's pre-bidding step, after saving course rankings and navigating to other screens (like the profile), then returning to the bidding page, the saved course rankings are not visible in the "Your Ranked Courses" list. The user has to reload the entire page to see the saved changes. However, the data is correctly saved on the server, as clicking into "details" view still shows the saved ranks.

## Root Cause
The `activeCampaign` query in `bidding-web` (which fetches the campaign data, including `ranked_courses`) uses the global `staleTime` of 5 minutes and `refetchOnMount: false`. 
When a student saves their rankings, the `useSubmitRankings` mutation invalidates the `activeCampaign` query, marking it as stale. 
If the student navigates away (making the query inactive) and then navigates back (making it active again), React Query sees that `refetchOnMount` is `false` and does NOT re-fetch the data, even though it is marked as stale. Consequently, the user sees the old (cached) data which doesn't include their "just saved" changes.

## Scenarios

### Scenario 1: Saved rankings visible after navigation
**Given** a student is on the Pre-Bidding page for an active campaign
**And** they have added and ranked 3 courses
**When** they click "Save"
**Then** they see a "Ranking saved!" toast notification
**And** the `activeCampaign` query is invalidated in the background
**When** the student navigates to their Profile page
**And** they navigate back to the Bidding page
**Then** the "Your Ranked Courses" list should **immediately re-fetch** and **display** the 3 ranked courses correctly.

### Scenario 2: Rankings are persistent across multiple navigations
**Given** a student has saved rankings in the current module
**When** they navigate between multiple pages (Profile, Dashboard, then back to Bidding)
**Then** the Pre-Bidding page should **always** display the most up-to-date rankings from the server.

## Acceptance Criteria
- Bidding page course list is consistent with the server-side state after saving and navigating away/back.
- `activeCampaign` query is forced to re-fetch on mount if it's stale (which it is after any save mutation).
- `waitlistStatus` query also re-fetches on mount if stale for consistent behavior in the waitlist phase.
