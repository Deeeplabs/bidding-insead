## Why

Despite previous fixes to `DateHelper::toIso()`, `DateHelper::parse()`, and the frontend date components, a **+1 hour offset** persists between the dates shown in the pre-bidding module configuration form and the dates displayed on both the PM Campaign list and the Student dashboard. Both the PM and Student are in the same timezone (GMT+8), yet the displayed start/end dates are consistently 1 hour ahead of the configured values (e.g., config shows `06:00`, display shows `07:00 GMT+8`).

This remaining issue is caused by **inconsistent timezone handling across multiple backend code paths**:

1. **Phase Activation Overwrites with Server-Local Time**: `CampaignPhaseService::activatePhase()` uses `$now = new \DateTime()` (server timezone: Europe/Paris, CET/CEST) and overwrites both the `phase_config.start_date` DATETIME column via `setStartDate($now)` and the JSON `module_config.start_date`/`actual_start_date` via `$now->format(\DateTime::ATOM)`. This replaces UTC-correct values with Europe/Paris local-time values, which `DateHelper::toIso()` then mislabels as UTC.
2. **Campaign-Level Date Save Without `DateHelper::parse()`**: `CampaignService` line 886 saves campaign start/end dates using `new \DateTime($data['start_date'])` directly, bypassing the UTC-safe `DateHelper::parse()` method.
3. **`new \DateTime()` Used Throughout for `$now` Comparisons**: Active-range checks (e.g., `$now >= $startDate && $now <= $endDate`) use `new \DateTime()` (server local) against Doctrine DateTime objects whose timezone context is ambiguous (Doctrine labels UTC DB values as Europe/Paris).
4. **Data Source Mismatch Between Config Form and Display**: The admin config DatePicker reads `module_config.start_date` from the JSON column (which may hold the original UTC ISO string or an activation-overwritten ATOM string). The student API and PM tooltip read from the DATETIME column through `DateHelper::toIso()`. When the activation flow overwrites only one of these sources, they diverge.

## What Changes

- **Fix `CampaignPhaseService.php` (Backend)**: Ensure `activatePhase()` and `closePhase()` use explicit UTC `DateTime` objects (`new \DateTime('now', new \DateTimeZone('UTC'))`) when setting `startDate`/`endDate` on phase configs, and use `DateHelper::toIso()` consistently for JSON config values (`actual_start_date`, `start_date`, etc.).
- **Fix `CampaignService.php` (Backend)**: Replace `new \DateTime($data['start_date'])` with `DateHelper::parse($data['start_date'])` for campaign-level start/end dates.
- **Fix `DateHelper.php` (Backend)**: Add a `utcNow()` helper to centralize UTC-safe `$now` creation. Ensure `toIso()` accounts for the possibility that the underlying DateTime has a non-UTC timezone by always converting to UTC before extracting the string.
- **Fix `is_active` Comparisons (Backend)**: Standardize all `$now` instantiations in active-range checks (Campaign entity, CampaignService, mappers) to use `DateHelper::utcNow()` so comparisons are consistently in UTC.
- **Fix `AdminCampaignDetailService.php` (Backend)**: Ensure the PM dashboard detail service formats phase dates through `DateHelper::toIso()`.

## Capabilities

### Modified Capabilities

- **Accurate Bidding Round Time Display**: Both PM and Student portals display start/end times that exactly match the configured values in the pre-bidding module configuration, regardless of server timezone or DST state.
- **Consistent Activation Timestamps**: When a PM manually starts or stops a phase, the recorded timestamps are UTC-correct, ensuring the displayed dates remain consistent with the original configuration.

## Impact

- **bidding-api**: `src/Helper/DateHelper.php` — Add `utcNow()` helper; harden `toIso()` to always convert to UTC.
- **bidding-api**: `src/Domain/Campaign/Campaign/CampaignPhaseService.php` — Use UTC DateTime in `activatePhase()`/`closePhase()` and `DateHelper::toIso()` for JSON config dates.
- **bidding-api**: `src/Domain/Campaign/Campaign/CampaignService.php` — Replace `new \DateTime(...)` with `DateHelper::parse()` for campaign dates; use `DateHelper::utcNow()` in `$now` comparisons.
- **bidding-api**: `src/Entity/Campaign.php` — Use `DateHelper::utcNow()` in `getCurrentActivePhaseDetails()` for `$now`.
- **bidding-api**: `src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapper.php` — Use `DateHelper::utcNow()`.
- **bidding-api**: `src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToModuleDetailDtoMapper.php` — Use `DateHelper::utcNow()`.
- **bidding-api**: `src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php` — Use `DateHelper::toIso()` for phase dates.
- **Affected roles**: Programme Managers and Students.
- **No migration required**: No database schema changes. Existing data with server-local-time values may need a one-time correction script.
- **Regression risk**: Low — changes enforce the UTC convention that was intended but not consistently applied.
