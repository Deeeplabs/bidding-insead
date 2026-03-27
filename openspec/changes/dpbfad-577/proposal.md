## Why

Students currently see their campaigns (active and historical) in the bidding view, but the ordering does not prioritize the most recent or recently-updated campaigns. When a PM creates a new campaign or edits an existing campaign's flow, the updated campaign does not appear at the top of the student's list. This forces students to scroll through older campaigns to find the most relevant ones, reducing usability.

## What Changes

- **API sorting**: Enforce consistent `COALESCE(updatedAt, createdAt) DESC` ordering on the student active-campaigns endpoint so newly created or recently edited campaigns always appear first.
- **Frontend sorting**: Apply client-side sort on the campaigns list in bidding-web to ensure latest campaigns render at the top, even if the API response order changes due to pagination or caching.
- **Visual separation**: Add clear visual grouping in the student bidding view to distinguish active campaigns from completed/historical campaigns while maintaining the "latest first" order within each group.

## Capabilities

### New Capabilities
- `student-campaign-ordering`: Sorting logic for student-facing campaign lists ensuring latest (newly created or recently updated) campaigns appear at the top, with clear active vs. historical grouping.

### Modified Capabilities

## Impact

- **bidding-api**: `StudentActiveCampaignController`, `ActiveCampaignService`, `CampaignRepository` — sorting/ordering logic for student campaign queries.
- **bidding-web**: `ActiveRoundCard.tsx`, `use-dashboard-stats.ts`, `campaign.queries.ts`, `phase.util.ts` — frontend sorting and visual grouping of campaigns.
- No database migration required — sorting uses existing `createdAt`/`updatedAt` fields on the Campaign entity.
- No breaking API changes — only the ordering of results within the existing response shape changes.
- No webhook or external integration impact.
