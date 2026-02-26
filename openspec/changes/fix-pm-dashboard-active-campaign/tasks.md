## 1. Update Domain Logic

- [x] 1.1 In `bidding-api/src/Domain/Campaign/Campaign/CampaignService.php` (`listCampaigns` method), map `$status === 'open'` to also apply `andWhere('c.endDate >= :now')` directly to the `$countQb` QueryBuilder.
- [x] 1.2 In `bidding-api/src/Repository/CampaignRepository.php` (`findOpenCampaigns` method), append the `endDate >= :now` condition to strictly enforce the "open campaigns" query.

## 2. Testing and Validation

- [x] 2.1 Verify via `GET /v2/api/campaigns` that `meta.total` perfectly matches the paginated logic limit output for active running campaigns, checking edge cases where endDate has passed.
- [x] 2.2 Verify via the dashboard stats API that `active_campaigns.count` accurately ignores the campaigns that successfully bypassed the `endDate` condition.
