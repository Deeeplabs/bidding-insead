## Root Cause Analysis

Previous fixes addressed the 8-hour shift (frontend double-parsing) and DST duration gaps (`DateHelper::toIso` reinterpretation, `DateHelper::parse`). However, a **consistent +1 hour offset** remains between the pre-bidding module configuration and the displayed dates on both PM Campaign list and Student dashboard.

### 1. Phase Activation Overwrites Dates with Server-Local Time
`CampaignPhaseService::activatePhase()` (line 759-762) executes:
```php
$now = new \DateTime(); // Europe/Paris server timezone (CET=UTC+1 / CEST=UTC+2)
$moduleConfig['actual_start_date'] = $now->format(\DateTime::ATOM); // e.g. "2026-03-30T07:00:00+02:00"
$moduleConfig['start_date'] = $now->format(\DateTime::ATOM);        // overwrites original UTC ISO
$phaseConfig->setStartDate($now);                                    // saves Paris local time to DATETIME column
```
When `DateHelper::toIso()` later reads the DATETIME column, it extracts the naive string (which is now Paris local time, not UTC) and mislabels it as UTC, producing a +1h (CET) or +2h (CEST) shift.

### 2. Campaign-Level Dates Bypass `DateHelper::parse()`
`CampaignService` line 886 saves campaign dates directly:
```php
$campaign->setStartDate(new \DateTime($data['start_date']));
```
This uses PHP's default timezone (Europe/Paris) for strings without explicit timezone info, potentially storing Paris-local values instead of UTC.

### 3. Inconsistent `$now` Timezone in Active-Range Checks
Throughout the codebase, `$now = new \DateTime()` creates a Europe/Paris DateTime. When compared against Doctrine DateTime objects from DATETIME columns (which Doctrine also labels as Europe/Paris, even though the underlying values are UTC), the comparisons happen to work. But when dates are written via the activation flow in Paris time, subsequent reads via `toIso()` produce incorrect UTC strings.

### 4. Data Source Divergence
- **Admin config form**: reads `module_config.start_date` from JSON column (original UTC ISO string from frontend `dayjs.toISOString()`)
- **Admin tooltip**: reads `module_config.start_date` from JSON (which may be overwritten by activation with ATOM-format Paris time)
- **Student API**: reads from DATETIME column via `DateHelper::toIso()`

After phase activation, the DATETIME column holds Paris local time while the config form may still show the original UTC-derived value, creating the observed +1h mismatch.

## Proposed Design

### 1. `DateHelper.php`: Add `utcNow()` and Harden `toIso()`

**`utcNow()`**: Centralized UTC-safe "now" factory:
```php
public static function utcNow(): \DateTime
{
    return new \DateTime('now', new \DateTimeZone('UTC'));
}
```

**`toIso()`**: Instead of only reinterpreting the naive string, explicitly convert to UTC first to handle DateTimes that may have a non-UTC timezone:
```php
public static function toIso(?\DateTimeInterface $date): ?string
{
    if ($date === null) return null;
    $utc = \DateTime::createFromInterface($date);
    $utc->setTimezone(new \DateTimeZone('UTC'));
    return $utc->format('Y-m-d\TH:i:s.000\Z');
}
```
**Critical change**: This correctly converts Paris-time DateTimes to UTC. For dates saved via `DateHelper::parse()` (which sets timezone to UTC), `setTimezone(UTC)` is a no-op. For dates from `new \DateTime()` (Europe/Paris), it properly shifts the hours.

**However**, this approach has a prerequisite: dates stored in the DB as UTC naive strings (from `DateHelper::parse()`) will be read by Doctrine as Europe/Paris, and `setTimezone(UTC)` will **shift them incorrectly**. Therefore, we must also ensure Doctrine's DateTime objects carry the correct timezone. The solution is to set PHP's default timezone to UTC application-wide via Symfony kernel boot, so Doctrine creates DateTimes in UTC.

### 2. Set Application Default Timezone to UTC
In the Symfony kernel or a boot listener, set:
```php
date_default_timezone_set('UTC');
```
This ensures:
- `new \DateTime()` produces UTC by default
- Doctrine reads DATETIME columns as UTC
- `toIso()` can safely use `setTimezone(UTC)` (no-op for UTC DateTimes)
- All `$now` comparisons are in UTC

### 3. Fix `CampaignPhaseService.php`
Replace all `$now = new \DateTime()` with `$now = DateHelper::utcNow()`. Use `DateHelper::toIso($now)` instead of `$now->format(\DateTime::ATOM)` for JSON config values.

### 4. Fix `CampaignService.php`
Replace `new \DateTime($data['start_date'])` with `DateHelper::parse($data['start_date'])` for campaign start/end dates.

### 5. Standardize `$now` Across All Services
Replace `new \DateTime()` with `DateHelper::utcNow()` in:
- `Campaign.php::getCurrentActivePhaseDetails()`
- `CampaignToActiveBiddingRoundDtoMapper.php`
- `CampaignToModuleDetailDtoMapper.php`
- `CampaignService.php` (all `$now`/`$today` usages)

## Affected Files Summary
| File Path | Impact | Rationale |
|-----------|--------|-----------|
| `bidding-api/src/Helper/DateHelper.php` | Backend | Add `utcNow()`; revise `toIso()` to use proper UTC conversion |
| `bidding-api/src/Kernel.php` or boot listener | Backend | Set `date_default_timezone_set('UTC')` application-wide |
| `bidding-api/src/Domain/Campaign/Campaign/CampaignPhaseService.php` | Backend | Use `DateHelper::utcNow()` and `DateHelper::toIso()` in activation/close flows |
| `bidding-api/src/Domain/Campaign/Campaign/CampaignService.php` | Backend | Use `DateHelper::parse()` for campaign dates; `DateHelper::utcNow()` for `$now` |
| `bidding-api/src/Entity/Campaign.php` | Backend | Use `DateHelper::utcNow()` in `getCurrentActivePhaseDetails()` |
| `bidding-api/src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapper.php` | Backend | Use `DateHelper::utcNow()` |
| `bidding-api/src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToModuleDetailDtoMapper.php` | Backend | Use `DateHelper::utcNow()` |
| `bidding-api/src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php` | Backend | Ensure `DateHelper::toIso()` for all phase date outputs |

## Risk Assessment
- **Application-Wide UTC Default**: Setting `date_default_timezone_set('UTC')` is the most impactful change. All existing `new \DateTime()` calls will now produce UTC instead of Europe/Paris. This is the CORRECT behavior for a multi-timezone application but must be tested thoroughly.
- **Backward Compatibility**: Dates already stored as UTC naive strings will now be correctly interpreted. Dates stored as Paris local time (from activation flow) will shift by 1-2 hours after the fix — these are the incorrect values being corrected.
- **Existing `DateHelper::toIso()` Reinterpretation**: The current "extract naive string, label as UTC" approach becomes unnecessary once Doctrine reads in UTC. The new `setTimezone(UTC)` approach is cleaner and handles both UTC and non-UTC DateTimes correctly.
