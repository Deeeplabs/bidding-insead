# Fix: UTC Timezone Standardization - Backend Implementation (Phase 2)

## Problem
Jira: [DPBFAD-919](https://insead.atlassian.net/browse/DPBFAD-919)

Phase 1 fixed the student-facing timezone display by reinterpreting naive `DATETIME` strings as UTC. However, a persistent **+1 hour offset** remained because the server's default timezone (`Europe/Paris`) was still applied by PHP and Doctrine whenever `new \DateTime()` was used. This caused:
- `CampaignPhaseService::activatePhase()` to overwrite `start_date`/`end_date` in JSON config with server-local time (`\DateTime::ATOM` format) instead of UTC.
- `CampaignService` to bypass `DateHelper::parse()` when saving campaign dates, using `new \DateTime($data['start_date'])` which inherited the server timezone.
- Inconsistent `new \DateTime()` usage across entities, mappers, and services producing Europe/Paris timestamps instead of UTC.

### 3. Capital (Bid Points) Check Disabled
`AddDropValidator::validateBidPoints()` had correct logic but was never called in `AddDropService::submitAddDrop()`. This loophole allowed submissions resulting in negative bid point balances.

### 1. `Kernel.php` — Set Application Default Timezone to UTC
- Added `date_default_timezone_set('UTC')` in `Kernel::boot()` so all `new \DateTime()` calls and Doctrine `DATETIME` reads default to UTC.

### 2. `DateHelper.php` — Add `utcNow()` and Revise `toIso()`
- Added `DateHelper::utcNow()` — returns `new \DateTime('now', new \DateTimeZone('UTC'))` for explicit UTC intent.
- Revised `toIso()` to use `DateTime::createFromInterface()` + `setTimezone(UTC)` instead of naive string reinterpretation, now that the app timezone is UTC.

### 3. `CampaignPhaseService.php` — Fix Activation Flow
- Replaced all `new \DateTime()` with `DateHelper::utcNow()`.
- Replaced all `new \DateTime($dateString)` with `DateHelper::parse($dateString)`.
- Changed `$now->format(\DateTime::ATOM)` to `DateHelper::toIso($now)` for JSON config values (`actual_start_date`, `actual_end_date`, `start_date`, `end_date`, `expected_start_date`, `expected_end_date`).

### 4. `CampaignService.php` — Fix Campaign Date Save
- Replaced `new \DateTime($data['start_date'])` and `new \DateTime($data['end_date'])` with `DateHelper::parse()`.
- Replaced all `new \DateTime()` (timestamps, status checks, query parameters) with `DateHelper::utcNow()`.
- Replaced `new \DateTime($phasePrevious['end_date'])` with `DateHelper::parse()` for phase completion checks.

### 5. `Campaign.php` Entity — Standardize `$now`
- Replaced all `new \DateTime()` with `DateHelper::utcNow()` in `getCurrentActivePhaseDetails()`, `getCurrentActivePhaseDetailsByModule()`, `getAutoDetectedCurrentModuleSequence()`, `computeDynamicStatus()`, `getPhaseBiddingPrebidding()`, and the constructor.

### 6. `CampaignToActiveBiddingRoundDtoMapper.php` — Standardize Mapper
- Replaced all `new \DateTime()` with `DateHelper::utcNow()` for deadline comparisons (`is_passed_deadline`), countdown calculations, and module active-state checks.

### 7. `CampaignToModuleDetailDtoMapper.php` — Standardize Mapper
- Replaced all `new \DateTime()` with `DateHelper::utcNow()` for deadline comparisons, module status evaluation, `isPhaseActive()`, and countdown calculations.

### 8. `AdminCampaignDetailService.php` — Fix Admin Dashboard
- Replaced all `new \DateTime()` with `DateHelper::utcNow()` for phase status computation, deadline query parameters, and class filtering.
- Replaced `new \DateTime($activePhase['start_date'])` / `new \DateTime($activePhase['end_date'])` with `DateHelper::parse()`.
- Replaced `new \DateTime($phasePrevious['end_date'])` with `DateHelper::parse()` in final enrollment detail.

## Impact
- **No Database Migrations**: All changes are in the interpretation/application layer.
- **Eliminates DST Drift**: With the app timezone forced to UTC, Doctrine reads and writes naive `DATETIME` columns as UTC. No more +1h offset from Europe/Paris DST.
- **Consistent Activation Timestamps**: `activatePhase()` now writes UTC ISO strings to JSON config, matching what the PM configured.
- **Backward Compatible**: `DateHelper::utcNow()` is a drop-in replacement for `new \DateTime()` and `DateHelper::parse()` for `new \DateTime($string)`.

## Verification Steps
1. Configure dates in PM portal and verify they match exactly in Student portal (no +1h shift).
2. Activate/close a phase and verify `actual_start_date`/`actual_end_date` in the JSON config are UTC.
3. Cross a DST boundary (e.g. March 28–30) and verify no drift in countdown timers.
4. Verify campaign CRUD operations (create, update, duplicate, delete) preserve correct dates.
5. Verify admin dashboard phase status (OPEN/COMPLETED/CLOSE) is computed correctly.
