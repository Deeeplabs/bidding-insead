## Root Cause Analysis

The current time display shift and duration discrepancies (like the missing 1 hour resulting in "1d 22h" instead of expected "1d 23h" during DST) are caused by fundamentally flawed handling of naive `DATETIME` columns by Doctrine combined with "Fake UTC" logic:

### 1. Database Naive UTC Drift (Backend)
When the backend saves `2026-03-27 17:00:00` (which is meant to be 17:00:00 UTC) info MySQL's `DATETIME` column, the timezone context is lost. When Doctrine retrieves this timestamp, it instantiates a `DateTime` object using the server's default timezone (Europe/Paris). Thus, the internal representation becomes `2026-03-27 17:00:00 Europe/Paris`, which is actually `16:00:00 UTC` (or `15:00:00 UTC` if Paris is in DST). This causes 1 or 2 hours to be permanently lost from the actual intended UTC time.

### 2. Incorrect UTC Conversion in `DateHelper`
The previous attempt to fix `DateHelper::toIso` called `$date->setTimezone(new \DateTimeZone('UTC'))`. This shifted the `17:00:00 Europe/Paris` back to `16:00:00 UTC`, preserving the DST/drift bug and passing the 1-2 hour offset down to the frontend, breaking expected durations (the "1d 23h" gap).

### 3. Frontend Double-Parsing Error
In `bidding-web`, components were maintaining a pattern of stripping timezone context during formatting (`useDate().format(date)`), and then re-parsing the local string as UTC, causing further 8-hour offsets in SGT.

## Proposed Design

The solution requires fully eliminating timezone drift by carefully defining the "Source of Truth" boundaries:

### Backend Refactor: `DateHelper.php` Reinterpretation
- **`toIso()`**: Instead of shifting the `DateTime` Object's timezone to UTC (which modifies the hours), reinterpret the underlying naive string from Doctrine as UTC:
  ```php
  $naiveString = $date->format('Y-m-d H:i:s');
  $utc = new \DateTime($naiveString, new \DateTimeZone('UTC'));
  return $utc->format('Y-m-d\TH:i:s.000\Z');
  ```
- **`parse()`**: Add robust parsing for ISO strings to ensure the newly instantiated `DateTime` object always has its explicit timezone set to UTC before passing it off to Doctrine (which will then safely save the `$date->format('Y-m-d H:i:s')` naive string).

### Frontend Refactor: Maintaining `Date` Context
- In `CollapsePanelExtra.tsx` and `HeaderSection.tsx`, maintain the `Date` object returned by the initial `parseUtc` call and bypass any naive string formatting.
- Pass the `Date` object directly to `getPhaseStatus`, `formatTimeRemaining`, and `getPhaseDisplayText`.

## Affected Files Summary
| File Path | Impact | Rationale |
|-----------|--------|-----------|
| `bidding-api/src/Helper/DateHelper.php` | Backend | Fix naive UTC reinterpretation in `toIso` and ensure strict UTC in `parse()` |
| `bidding-api/src/Controller/Api/Student/Campaign/StudentActiveCampaignController.php` | Backend | Use `toIso` and `parse()` for consistency |
| `bidding-api/src/Domain/Campaign/Campaign/CampaignService.php` | Backend | Ensure UTC alignment on campaign save |
| `bidding-api/src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToModuleDetailDtoMapper.php` | Backend | Use `DateHelper::parse()` for ISO string support |
| `bidding-web/src/features/bidding/components/shared/bidding-card/CollapsePanelExtra.tsx` | Frontend | Maintain `Date` objects to avoid UI timezone issues |
| `bidding-web/src/features/bidding/components/bid-submission/HeaderSection.tsx` | Frontend | Maintain `Date` objects |

## Risk Assessment
- **Zero-Shift Migration**: Reinterpreting the database values directly as UTC ensures any globally configured timestamps immediately act identically for all regions, perfectly restoring backward compatibility for expected countdown durations across DST periods.
- **Backward Compatibility**: `parseUtc` in frontend is idempotent for `Date` objects.
