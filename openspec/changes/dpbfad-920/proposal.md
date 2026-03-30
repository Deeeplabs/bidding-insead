# Proposal - Fix Bidding Page Ranking Sync After Navigation

## Why
When a student ranks courses in the pre-bidding phase and saves them, the changes are not reflected if they navigate away (e.g., to the profile page) and back to the bidding page. However, the data is correctly saved on the server, as evidenced by the "details" view showing the correct rankings. This discrepancy is caused by the `activeCampaign` query in the web frontend having a long `staleTime` (5 minutes) and `refetchOnMount: false` by default, which prevents the page from re-fetching the updated campaign data when the user navigates back to it, even after the data has been invalidated by a successful save mutation.

## What Changes
- Update the `activeCampaign` query in `campaign.queries.ts` to include `refetchOnMount: true`.
- Update the `waitlistStatus` query in `campaign.queries.ts` to include `refetchOnMount: true` for consistency and to avoid similar synchronization issues.
- Ensure that `useSubmitRankings` and other related mutations correctly invalidate relevant queries to trigger a re-fetch when `refetchOnMount: true` is set.

## Capabilities
### New Capabilities
- `campaign-data-sync-on-mount`: Automatically re-fetch campaign details and waitlist status when the student navigates back to the bidding or waitlist pages, ensuring they always see the latest saved state.

## Impact
- **bidding-web**:
  - `src/lib/api/queries/campaign.queries.ts`: Update `activeCampaign` and `waitlistStatus` query options.
- **Entities affected**: None (frontend-only fix).
- **Migration**: None.
- **Frontend changes**:
  - Improved data consistency on the student portal.
  - Eliminated the bug where saved rankings appear "gone" after navigation.
- **Backward compatibility**: No impact on existing functionality; only affects how often data is refreshed from the server.
