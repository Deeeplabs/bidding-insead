## Implementation: `bidding-api` (Backend)

- [x] **Task 1: Fix `DateHelper.php`**
  - [x] Modify `toIso()` in `src/Helper/DateHelper.php` to convert dates to UTC before formatting.
  - [x] Add `parse()` to `src/Helper/DateHelper.php` to robustly handle ISO and naive MySQL strings as UTC.
  
- [x] **Task 2: Standardize `StudentActiveCampaignController.php`**
  - [x] Replace manual `.format('Y-m-d H:i:s')` with `DateHelper::toIso()` for consistent UTC output.
  - [x] Use `DateHelper::parse()` to handle potential ISO input strings.

- [x] **Task 3: Refactor Backend Mapping and Services**
  - [x] Update `CampaignToModuleDetailDtoMapper.php` to use `DateHelper::parse()` for ISO string support.
  - [x] Update `CampaignService.php` to ensure UTC interpretation during campaign save.

## Implementation: `bidding-web` (Frontend)

- [x] **Task 4: Refactor `CollapsePanelExtra.tsx`**
  - [x] Locate `src/features/bidding/components/shared/bidding-card/CollapsePanelExtra.tsx`.
  - [x] Modify to use `parseUtc` directly and avoid intermediate local formatting.
  
- [x] **Task 5: Refactor `HeaderSection.tsx`**
  - [x] Locate `src/features/bidding/components/bid-submission/HeaderSection.tsx`.
  - [x] Modify to use `parseUtc` directly and preserve `Date` context.

## Verification

- [x] **Task 6: Post-Implementation Verification**
  - [x] Verify that the API returns true UTC (ending in Z) for campaign dates.
  - [x] Verify that PM configurations in SGT correctly match the student portal display.
  - [x] Verify no regression on DST transition days (e.g., March 29).
