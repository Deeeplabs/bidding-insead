# Fix: Student Timezone Display - Backend Implementation

## Problem
Jira: [DPBFAD-919](https://insead.atlassian.net/browse/DPBFAD-919)

The Student Portal was displaying incorrect times (such as round durations being `1d 23h` instead of exactly `48h`) because of a combination of "Fake UTC" strings and Doctrine timezone drift. When a PM configured a round from March 28 to March 30, the naive `DATETIME` saved in MySQL was loaded by Doctrine using the server's default timezone (`Europe/Paris`). The previous fix attempted to convert this `DateTime` object to UTC using `setTimezone()`, which inadvertently shifted the time backward by an hour during daylight saving time transitions, causing countdowns to clip an hour.

### 3. Capital (Bid Points) Check Disabled
`AddDropValidator::validateBidPoints()` had correct logic but was never called in `AddDropService::submitAddDrop()`. This loophole allowed submissions resulting in negative bid point balances.

### 1. `bidding-api/src/Helper/DateHelper.php`
- Refactored `toIso()` to completely avoid shifting hours across DST boundaries. It now extracts the naive string from the Doctrine-provided `DateTime` (i.e. `Y-m-d H:i:s`) and re-instantiates it directly mapped to a `UTC` timezone.
- Updated `DateHelper::parse()` to robustly handle incoming ISO strings and explicitly map them to UTC so Doctrine persists the exact requested time in the naive `DATETIME` columns without any server offset drift.

### 2. Standardized Controllers & Mappers
- Used `DateHelper::toIso()` and `DateHelper::parse()` in `StudentActiveCampaignController` and `CampaignToModuleDetailDtoMapper` to ensure that standard UTC strings (ending in `Z`) are constantly sent to the frontend, representing the exact original time configured by the PM.

## Impact
- **No Database Migrations**: This change fixes the interpretation layer without modifying database schema.
- **Eliminates DST Drift**: Timers no longer incorrectly display `1d 23h` when a 48 hour round is scheduled passing over a DST change, since the backend now preserves the absolute intent of the configured times as identical UTC representations regardless of server timezone.

## Verification Steps
1. Create a campaign crossing a DST boundary (e.g. March 28 to March 30).
2. Perform a `GET` request to `/student/active-campaigns/{campaignId}`.
3. Verify that `start_date`, `end_date`, and `course_class_deadline` strings mathematically equate to the exact naive database times without dropping an hour.
