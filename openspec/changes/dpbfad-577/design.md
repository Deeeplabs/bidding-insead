## Context

Students see their campaigns (active and historical) in the bidding view via the `/student/active-campaigns` endpoint. The current backend sorts eligible campaigns by `createdAt DESC` only. When a PM edits a campaign's flow (e.g., changes phases, updates modules), the campaign's `updatedAt` timestamp changes but `createdAt` does not — so the edited campaign stays in its original position rather than moving to the top of the list.

The frontend (`ActiveRoundCard.tsx`) renders campaigns in the order received from the API without any additional sorting at the campaign level. It only sorts bidding rounds within each campaign by `sequence_order`.

**Current sorting flow:**
1. `ActiveCampaignService::getActiveBiddingRounds()` fetches eligible campaigns for a student
2. `usort()` sorts by `createdAt DESC` (newly created first, but recently edited campaigns don't rise)
3. Frontend receives paginated campaigns and renders in API order
4. `filterCampaignsVisibleRounds()` only sorts rounds within each campaign, not campaigns themselves

## Goals / Non-Goals

**Goals:**
- Latest campaigns (newly created OR recently updated) appear at the top of the student's campaign list
- Sorting is automatic — no student action or page refresh required
- Works consistently for both active and historical campaigns
- Active campaigns visually separated from completed campaigns

**Non-Goals:**
- User-configurable sort order (students cannot choose sort criteria)
- Admin-side campaign list ordering changes (this is student-facing only)
- Changing the pagination behavior or infinite scroll mechanics
- Adding new API response fields or changing existing response shape

## Decisions

### 1. Sort by `COALESCE(updatedAt, createdAt) DESC` in `ActiveCampaignService`

**Change:** In `ActiveCampaignService::getActiveBiddingRounds()`, modify the `usort()` comparator to use `updatedAt` when available, falling back to `createdAt`.

**Rationale:** This is the simplest change — one comparator update in a single service method. The Campaign entity already has both `createdAt` and `updatedAt` fields. When a PM edits a campaign, `updatedAt` is automatically set by Doctrine's lifecycle callbacks, causing the campaign to rise to the top.

**Alternative considered:** Adding a separate `lastActivityAt` field — rejected because it requires a migration and the existing timestamps already capture the needed behavior.

### 2. Add client-side sort in `filterCampaignsVisibleRounds()` as defense-in-depth

**Change:** Before mapping individual campaign rounds, sort the campaigns array itself by the most recent date (using `start_date` or response timestamps).

**Rationale:** The API already returns campaigns in the correct order, but adding a frontend sort ensures correct ordering even across paginated infinite-scroll pages that may arrive at different times. It also protects against any future API changes.

### 3. Group campaigns by active vs. completed status in `ActiveRoundCard.tsx`

**Change:** In the `ActiveRoundCard` component, partition campaigns into two groups: active (campaigns with ongoing/upcoming phases) and completed (all phases ended). Render active campaigns first, then completed, each with a section header.

**Rationale:** This directly addresses acceptance criterion #5 (UI clearly separates campaigns). Within each group, the "latest first" sort order is maintained. The grouping uses existing phase status logic from `getPhaseStatus()` in `phase.util.ts`.

**Alternative considered:** Separate tabs for active/completed — rejected because it hides information and changes the existing UX pattern (single scrollable list with accordion).

## Risks / Trade-offs

- **[Low] Sort stability across pages**: Infinite scroll loads campaigns in pages. If a campaign is updated between page loads, it could theoretically appear twice (once in a new page, once cached in an earlier page). → Mitigation: React Query's `gcTime` (5 min) and polling (5s) will refresh the cache. The existing `maxPages` limit prevents unbounded growth.

- **[Low] Behavioral change for existing users**: Students accustomed to the current ordering will see campaigns in a different order. → Mitigation: The new order is objectively more useful (latest first). No feature flag needed since this is a pure UX improvement with no data risk.

- **[None] No migration required**: Uses existing `createdAt`/`updatedAt` fields — no schema changes needed.
