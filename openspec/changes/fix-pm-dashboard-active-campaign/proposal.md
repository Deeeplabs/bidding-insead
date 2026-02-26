## Why

The PM Dashboard displays "Active Campaigns: 68 Running now" in the header and the pagination label shows "Showing 61 - 68 of 68 campaigns", but the table only displays 4 items on the last page. The manual count does not match the header stats or the total count in the pagination label. This discrepancy is caused by missing `endDate >= now` filtering in the count query for pagination and the PM dashboard stats query, leading to an incorrect inflation of active campaigns in the total count, while the actual paginated query correctly filters them out.

## What Changes

To fix this issue, two changes will be applied:
1. In `src/Domain/Campaign/Campaign/CampaignService.php` (`listCampaigns` method), the `$countQb` query builder will be updated to include the `endDate >= :now` condition when filtering by `status === 'open'`, ensuring the total count matches the data returned.
2. In `src/Repository/CampaignRepository.php` (`findOpenCampaigns` method), the query will be updated to include the `endDate >= :now` condition. This ensures that the PM Dashboard stats (Active Campaigns) accurately counts only currently open and running campaigns.

## Capabilities

### New Capabilities

### Modified Capabilities
- `pm-dashboard`: Adjust the active campaign fetching and counting logic to accurately align with the actual data shown in the active campaigns table data (with proper `endDate` validations).

## Impact

- `CampaignService.php`: Fixed total count calculations in pagination for `/api/campaigns`.
- `CampaignRepository.php`: Fixed active campaign counts for the `PMDashboardStatsService` dashboard header.
- No database migrations or API response structure changes are required.
