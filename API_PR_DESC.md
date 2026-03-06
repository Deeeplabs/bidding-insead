# Fix Switch Calendar View — Duplicate Module Entries

## Problem

When a Programme Manager (PM) navigates to **Switch → Dashboard → Calendar View**, duplicate module entries appear on the Gantt-chart timeline. For example, "Module 1: IN_PERSON FBL" shows up multiple times under the same programme row, each containing the same exact courses.

**Root cause:** The `getCalendar()` method in `PM\FlexSwitchService` deduplicates calendar items using the key `deliveryMode_promotionPeriodId_campusId`. The `promotionPeriodId` is the **database entity ID** (unique per promotion×period row), not the **logical period number** (e.g., "1", "2"). When a class has multiple `classPromotion` records linked to different promotions (e.g., J25 and D25 cohorts) that share the same period number but have different `promotionPeriodId` values, each generates a separate calendar item — despite being labeled identically — causing the duplicates.

Additionally, `ClassesRepository::findCalendarClassesByProgram()` fetched **all** CORE classes globally without filtering by the requested program, relying entirely on in-memory filtering. This loaded unnecessary data and increased processing time.

## Solution

Two targeted fixes in the API backend — no frontend changes required:

### Changes Made

**Modified File:**

1. **`src/Repository/ClassesRepository.php`**
   - **Method**: `findCalendarClassesByProgram(int $programId)`
   - Added joins through `classPromotions → promotion → program` to filter classes by program at the SQL level
   - Added `DISTINCT` to avoid duplicate class rows from the additional joins
   - This eliminates unnecessary data loading (previously fetched all CORE classes across all programs)

2. **`src/Service/FlexSwitch/PM/FlexSwitchService.php`**
   - **Method**: `getCalendar(int $programId)`
   - Changed the deduplication key from `$modeValue . '_' . $promotionPeriodId . '_' . $campusId` to `$modeValue . '_' . $period . '_' . $campusId`
   - Uses the logical period number (`$period` from `$promotionPeriod->getPeriod()`) instead of the entity ID (`$promotionPeriodId`)
   - This ensures modules with the same delivery mode, period number, and campus collapse into a single calendar item
   - Date range merging (min start_date / max end_date) continues to work correctly

## Impact

- **No API response shape changes**: The `FlexSwitchCalendarDto` structure is unchanged — only duplicate entries are eliminated
- **No frontend changes needed**: `bidding-admin` and `bidding-web` consume the same response format
- **No migration required**: No database schema changes
- **Performance improvement**: Repository query now filters by program at SQL level instead of loading all CORE classes globally
- **Backward compatible**: No breaking changes to existing consumers

## Verification

- [ ] PM admin panel: Switch → Dashboard → Calendar View shows no duplicate module rows
- [ ] Each module displays correct date ranges spanning all underlying classes
- [ ] Clicking a module opens the drawer with the correct course list
- [ ] API response `GET /flex-switch/calendar?program_id={id}` has no duplicate `module` labels within a group
