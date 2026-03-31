## Previously Implemented (Phase 1 — completed)

- [x] **Task 1: Fix `DateHelper.php` Naive UTC Drift**
  - [x] Modify `toIso()` to reinterpret the database naive string as UTC.
  - [x] Add `parse()` to handle incoming ISO strings with explicit UTC timezone.
  
- [x] **Task 2: Standardize `StudentActiveCampaignController.php`**
  - [x] Replace manual `.format('Y-m-d H:i:s')` with `DateHelper::toIso()`.
  - [x] Use `DateHelper::parse()` for inputs.

- [x] **Task 3: Refactor Backend Mapping and Services**
  - [x] Update `CampaignToModuleDetailDtoMapper.php` to use `DateHelper::parse()`.
  - [x] Update `CampaignService.php` phase config save to use `DateHelper::parse()`.

- [x] **Task 4: Refactor `CollapsePanelExtra.tsx`**
  - [x] Use `parseUtc` directly, avoid intermediate local formatting.
  
- [x] **Task 5: Refactor `HeaderSection.tsx`**
  - [x] Use `parseUtc` directly, preserve `Date` context.

## Implementation: Phase 2 — Fix +1h Offset (Backend)

- [ ] **Task 6: Set Application Default Timezone to UTC**
  - [ ] In `bidding-api/src/Kernel.php` boot method (or a Symfony event subscriber on `kernel.request`), add `date_default_timezone_set('UTC')` so all `new \DateTime()` calls and Doctrine DATETIME reads default to UTC.
  - [ ] Verify existing tests still pass after the timezone change.

- [ ] **Task 7: Add `DateHelper::utcNow()` Helper**
  - [ ] Add `utcNow()` static method to `src/Helper/DateHelper.php` that returns `new \DateTime('now', new \DateTimeZone('UTC'))`.
  - [ ] Revise `toIso()` to use `$date->setTimezone(new \DateTimeZone('UTC'))` before formatting, replacing the current naive-string reinterpretation approach (safe once the application default timezone is UTC).

- [ ] **Task 8: Fix `CampaignPhaseService.php` Activation Flow**
  - [ ] In `activatePhase()` (open flow): replace `$now = new \DateTime()` with `$now = DateHelper::utcNow()`.
  - [ ] Replace `$now->format(\DateTime::ATOM)` with `DateHelper::toIso($now)` for `module_config.actual_start_date`, `module_config.start_date`, and `module_config.expected_start_date`.
  - [ ] In `closePhase()` (close flow): same changes for `$now`, `actual_end_date`, `end_date`, `expected_end_date`.
  - [ ] Ensure `$phaseConfig->setStartDate($now)` and `setEndDate($now)` receive UTC DateTimes.

- [ ] **Task 9: Fix `CampaignService.php` Campaign Date Save**
  - [ ] Replace `$campaign->setStartDate(new \DateTime($data['start_date']))` (line ~886) with `$campaign->setStartDate(DateHelper::parse($data['start_date']))`.
  - [ ] Same for `$campaign->setEndDate(...)` (line ~890).
  - [ ] Replace all `$now = new \DateTime()` and `$today = new \DateTime()` with `DateHelper::utcNow()` throughout the file.

- [ ] **Task 10: Standardize `$now` in Campaign Entity and Mappers**
  - [ ] `src/Entity/Campaign.php` — `getCurrentActivePhaseDetails()` and `getCurrentActivePhaseDetailsByModule()`: replace `$now = new \DateTime()` with `DateHelper::utcNow()`.
  - [ ] `src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapper.php` — replace `$now = new \DateTime()` with `DateHelper::utcNow()` in `buildActiveBiddingRoundGroups()`, `buildModuleItem()`, `calculateCountdown()`.
  - [ ] `src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToModuleDetailDtoMapper.php` — replace `$now = new \DateTime()` with `DateHelper::utcNow()` in `buildModulesForBiddingRound()`, `isPhaseActive()`, `calculateCountdown()`.

- [ ] **Task 11: Fix `AdminCampaignDetailService.php`**
  - [ ] Ensure all phase date outputs use `DateHelper::toIso()`.
  - [ ] Replace `$now = new \DateTime()` with `DateHelper::utcNow()`.

## Verification

- [ ] **Task 12: Post-Implementation Verification**
  - [ ] **Config vs Display Match**: Configure a pre-bidding module with specific start/end times. Verify PM Campaign list tooltip and Student dashboard display EXACTLY the same times as configured (both in GMT+8).
  - [ ] **Activation Flow**: Manually start a phase via PM dashboard. Verify the `actual_start_date` in the response is correct UTC and displays correctly in all views.
  - [ ] **Cross-DST Verification**: Create a round crossing Europe/Paris DST boundary (e.g., 28 Mar to 30 Mar 2026). Verify countdown durations are exact and no 1-2 hour drift occurs.
  - [ ] **Regression Check**: Verify campaign CRUD operations, phase date saves, and `is_active` checks all work correctly with the UTC default timezone.
