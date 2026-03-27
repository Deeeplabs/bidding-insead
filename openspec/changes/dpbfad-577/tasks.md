## 1. Backend — Campaign Sorting

- [x] 1.1 Update `ActiveCampaignService::getActiveBiddingRounds()` in `bidding-api/src/Domain/Campaign/ActiveCampaign/ActiveCampaignService.php` to sort campaigns by `COALESCE(updatedAt, createdAt) DESC` instead of `createdAt DESC`. Add `id DESC` as a tiebreaker for deterministic ordering when dates are equal.
- [x] 1.2 Add/update PHPUnit test in `bidding-api/tests/Unit/Domain/Campaign/` to verify that campaigns are returned in `COALESCE(updatedAt, createdAt) DESC` order — covering cases: newly created campaign at top, recently updated campaign moves to top, campaign with null `updatedAt` uses `createdAt`, and same-date tiebreaker uses id DESC.

## 2. Frontend — Campaign Sorting

- [x] 2.1 Update `filterCampaignsVisibleRounds()` in `bidding-web/src/features/bidding/utils/phase.util.ts` to sort the campaigns array itself (not just rounds within campaigns) by latest date descending before mapping rounds. Use `start_date` or equivalent timestamp from the API response, falling back to `end_date`.
- [x] 2.2 Update `ActiveRoundCard.tsx` in `bidding-web/src/features/bidding/components/bidding-list/ActiveRoundCard.tsx` to partition campaigns into two groups: active (campaigns with ongoing/upcoming phases) and completed (all phases ended). Render active campaigns above completed, each with a section header ("Active Campaigns" / "Completed Campaigns"). Omit headers when only one group has campaigns. Maintain latest-first order within each group.

## 3. Verification

- [x] 3.1 Manually verify: create two campaigns for the same promotion, update the older one, confirm it moves to the top of the student bidding view. Verify both active and completed campaigns display in correct groups with correct ordering.
