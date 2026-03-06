## Why

The PM Switch Calendar View displays duplicate modules that contain the same exact courses. When a PM navigates to **Switch → Calendar View**, modules such as "Module 1: IN_PERSON FBL" appear multiple times under the same programme row, each showing identical course content.

**Root cause**: The `getCalendar()` method in `PM\FlexSwitchService` iterates over every `classPromotion` for each class. A single class can have multiple `classPromotion` records linked to different promotions (e.g., different year cohorts) that share the same promotion period number but have different `promotionPeriodId` values. The deduplication key (`$itemKey = $modeValue . '_' . $promotionPeriodId . '_' . $campusId`) uses the unique `promotionPeriodId` (entity ID), not the logical period number. This means two classPromotions with period number "1" but different `promotionPeriodId` values (e.g., period ID 10 vs period ID 20) produce separate calendar items — both labeled "Module 1: IN_PERSON FBL" — creating the duplicate.

Additionally, the repository method `findCalendarClassesByProgram()` fetches ALL CORE classes globally without filtering by the requested program, relying on in-memory filtering. This loads unnecessary data and increases the chance of generating duplicate entries.

## What Changes

Fix the calendar data aggregation to deduplicate modules by their logical identity (delivery mode + period number + campus) rather than by database entity IDs. Also optimize the repository query to filter by program at the database level.

## Capabilities

### New Capabilities

_None_

### Modified Capabilities

- `flex-switch-calendar`: Fix duplicate module grouping in PM calendar view by deduplicating on logical period number instead of promotion period entity ID, and filter classes by program in the repository query

## Impact

- **bidding-api**:
  - `App\Repository\ClassesRepository::findCalendarClassesByProgram()` — add program filter to SQL query via classPromotions → promotion → program join
  - `App\Service\FlexSwitch\PM\FlexSwitchService::getCalendar()` — change item deduplication key from `$modeValue . '_' . $promotionPeriodId . '_' . $campusId` to `$modeValue . '_' . $period . '_' . $campusId` (use period number instead of period entity ID)
- **bidding-admin**: No frontend changes needed — the data shape is unchanged, just the duplicates are eliminated
- **bidding-web**: No changes needed
- **No migration required**: No schema changes
- **No API response shape changes**: The `FlexSwitchCalendarDto` structure stays the same, only the duplicate entries are removed
- **No breaking changes**: Calendar items will simply be deduplicated
