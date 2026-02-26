## Context

Currently, the Programme Manager (PM) Dashboard has a display issue for active campaigns. The active campaign count displayed on the header ("overall stats") and the pagination details at the bottom of the list ("Showing X-Y of Z campaigns") are out of sync with the actual list of campaigns rendered. The queries fetching the total count for stats and pagination omit a constraint filtering for campaigns where `endDate >= :now` when checking for 'open' active campaigns, causing the total count to inflate and be incorrect compared to the accurately filtered active campaign list.

## Goals / Non-Goals

**Goals:**
- Fix the active campaign count calculation by enforcing the `endDate >= :now` constraint when computing the total open campaigns in both the dashboard stats and the paginated active campaign API list.
- Eliminate the mismatch in the pagination text against the visual table list.

**Non-Goals:**
- No changes to the core PM Dashboard layout or design.
- No changes to the database scheme or how campaign statuses are fundamentally stored.

## Decisions

- **Modify `CampaignService::listCampaigns`**: The count query inside this method handles pagination meta-data. It already filters by status, but will now additionally apply `c.endDate >= :now` when `$status === 'open'`, aligning the total count computation with the logic used for fetching the paginated rows.
- **Modify `CampaignRepository::findOpenCampaigns`**: This method fetches active campaigns specifically used for the dashboard header stats. We will introduce the `pc.endDate >= :now` constraint here to ensure strictly open and running campaigns are counted.

## Risks / Trade-offs

- **Risk:** Some campaigns may appear closed faster if their `endDate` is reached but the status was never updated. This is structurally correct behavior as a campaign beyond its end date shouldn't technically be 'Running now' but may present a sudden slight drop in counts directly following this fix deployment. This is an intended consequence and tradeoff for correct functionality.
