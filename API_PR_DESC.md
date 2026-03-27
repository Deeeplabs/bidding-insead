# Feature: Display Student Campaigns with Latest on Top

## Problem
Jira: https://insead.atlassian.net/browse/DPBFAD-577

Students see all their campaigns (active and historical) in the bidding view, but the list is sorted only by `createdAt DESC`. When a PM creates a new campaign or edits an existing campaign's flow (phases, modules), the updated campaign does not move to the top. Students must scroll through older campaigns to find the most relevant ones.

## Goal
- Sort student campaigns by latest activity (`updatedAt` or `createdAt`) so newly created or recently edited campaigns always appear first.

## Changes Made

### 1. `bidding-api/src/Domain/Campaign/ActiveCampaign/ActiveCampaignService.php`
- Updated the `usort()` comparator in `getActiveBiddingRounds()` to sort by `COALESCE(updatedAt, createdAt) DESC` instead of `createdAt DESC`.
- Added `id DESC` as a tiebreaker for deterministic ordering when dates are equal.
- When a PM edits a campaign, its `updatedAt` is automatically set by Doctrine lifecycle callbacks, causing it to rise to the top of the student's list.

### 2. `bidding-api/tests/Unit/Domain/Campaign/ActiveCampaign/ActiveCampaignServiceSortTest.php` (new)
- Added PHPUnit tests covering four sorting scenarios:
  - Newly created campaign appears first
  - Recently updated campaign moves to top
  - Campaign with null `updatedAt` falls back to `createdAt`
  - Same-date tiebreaker uses `id DESC`

## Impact
- **No migrations required** — Sorting uses existing `createdAt`/`updatedAt` fields on the Campaign entity.
- **No breaking API changes** — Only the ordering of results within the existing response shape changes.
- **No webhook or external integration impact.**

## Testing / Verification Steps
1. Open the student bidding view and verify campaigns appear with the most recently created/updated campaign at the top.
2. As a PM, edit an older campaign's flow (e.g., update phase config). Verify the campaign moves to the top of the student's list.
3. Run PHPUnit tests: `bin/phpunit tests/Unit/Domain/Campaign/ActiveCampaign/ActiveCampaignServiceSortTest.php`
