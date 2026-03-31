## Implementation: `bidding-api` (Backend)

- [x] **Task 1: Fix `DateHelper.php` Naive UTC Drift**
  - [x] Modify `toIso()` in `src/Helper/DateHelper.php` to reinterpret the database naive string as UTC: format the date as `Y-m-d H:i:s` first, then instantiate a new `DateTime` with `DateTimeZone('UTC')` before outputting the ISO string.
  - [x] Update `parse()` in `src/Helper/DateHelper.php` to robustly handle incoming ISO strings and explicitly set their explicit timezone to UTC so Doctrine saves the exact naive `Y-m-d H:i:s` value.
  
- [x] **Task 2: Standardize `StudentActiveCampaignController.php`**
  - [x] Replace manual `.format('Y-m-d H:i:s')` outputs with `DateHelper::toIso()` for consistent UTC string generation.
  - [x] Use `DateHelper::parse()` for any inputs.

- [x] **Task 3: Refactor Backend Mapping and Services**
  - [x] Update `CampaignToModuleDetailDtoMapper.php` to use `DateHelper::parse()` for ISO string support.
  - [x] Update `CampaignService.php` to ensure `startDate` and `endDate` are strictly instantiated via `DateHelper::parse()` on setup and save.

## Implementation: `bidding-web` (Frontend)

- [x] **Task 4: Refactor `CollapsePanelExtra.tsx`**
  - [x] Locate `src/features/bidding/components/shared/bidding-card/CollapsePanelExtra.tsx`.
  - [x] Modify to use `parseUtc` directly and avoid intermediate local formatting so that the Countdown accurately receives the absolute bounds without double-parsing.
  
- [x] **Task 5: Refactor `HeaderSection.tsx`**
  - [x] Locate `src/features/bidding/components/bid-submission/HeaderSection.tsx`.
  - [x] Modify to use `parseUtc` directly and preserve `Date` context.

## Verification

- [x] **Task 6: Post-Implementation Verification**
  - [x] **Countdown and Duration Verification**: Create a round intentionally crossing Europe/Paris DST boundary (e.g., 28 Mar to 30 Mar). Verify the frontend countdown accurately displays durations without dropping 1-hour ("1d 23h" vs "48h" verification).
  - [x] Verify that the API returns true UTC (ending in Z) representing the original naive time in the database without any drift caused by server offsets.
  - [x] Verify no regression on PM configuration in standard SGT regions perfectly persisting in exactly the intended timeframe.
