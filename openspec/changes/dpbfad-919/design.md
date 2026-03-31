## Root Cause Analysis

The current time display shift is caused by a failure to maintain a true UTC "Source of Truth" through the API:

### 1. Backend: "Fake UTC" Strings
In `bidding-api/src/Helper/DateHelper.php`, the `toIso` method was defined as:
```php
return $date->format('Y-m-d\TH:i:s.000\Z');
```
This appends a `Z` suffix manually without converting the `DateTime` object to the UTC timezone first. If the server is in a local timezone (e.g., Paris or SGT), the resulting string is a "Fake UTC" string (e.g., `"2026-03-27T01:20:00Z"` when 1:20 was actually the local time).

### 2. Frontend: Double-Parsing Error
In `bidding-web`, components followed this pattern:
1. `parseUtc(date_from_api)` -> Correct `Date` object (assuming API was true UTC).
2. `useDate().format(date)` -> Local string without TZ (e.g., `"2026-03-27 01:20:00"`).
3. Passed to utilities that call `parseUtc()` again.
4. `parseUtc()` interprets non-suffixed strings as UTC -> `2026-03-27T01:20:00Z`.
5. Final rendering adds local offset *again*, resulting in an 8-hour shift.

## Proposed Design

The solution is a coordinated fix to restore true UTC handling at both ends of the pipeline.

### Backend Refactor: `DateHelper.php`
- Convert all `DateTimeInterface` objects to `UTC` before formatting with the `Z` suffix.
- Ensure that `StudentActiveCampaignController` and other student-facing controllers use `toIso()` instead of manual `.format('Y-m-d H:i:s')`.

### Frontend Refactor: Maintaining `Date` Context
- In `CollapsePanelExtra.tsx` and `HeaderSection.tsx`, maintain the `Date` object returned by the initial `parseUtc` call.
- Pass the `Date` object directly to `getPhaseStatus` and `getPhaseDisplayText`, avoiding any intermediate string formatting that loses timezone context.

## Affected Files Summary
| File Path | Impact | Rationale |
|-----------|--------|-----------|
| `bidding-api/src/Helper/DateHelper.php` | Backend | Fix UTC conversion in `toIso` and add `parse()` |
| `bidding-api/src/Controller/Api/Student/Campaign/StudentActiveCampaignController.php` | Backend | Use `toIso` and `parse()` for consistency |
| `bidding-api/src/Domain/Campaign/Campaign/CampaignService.php` | Backend | Ensure UTC alignment on campaign save |
| `bidding-api/src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToModuleDetailDtoMapper.php` | Backend | Use `DateHelper::parse()` for ISO string support |
| `bidding-web/src/features/bidding/components/shared/bidding-card/CollapsePanelExtra.tsx` | Frontend | Maintain `Date` objects |
| `bidding-web/src/features/bidding/components/bid-submission/HeaderSection.tsx` | Frontend | Maintain `Date` objects |

## Risk Assessment
- **Zero-Shift Migration**: Since the backend now returns true UTC, all frontend components already using the (now correct) UTC strings will display correctly.
- **Backward Compatibility**: `parseUtc` is idempotent for `Date` objects, so passing objects instead of strings is safe.
