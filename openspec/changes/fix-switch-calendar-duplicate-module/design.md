## Context

The PM Switch Calendar View (`GET /flex-switch/calendar`) displays a Gantt-chart timeline of modules grouped by programme and campus. The data is built by `PM\FlexSwitchService::getCalendar()`, which:

1. Fetches all CORE classes via `ClassesRepository::findCalendarClassesByProgram()` (currently **unfiltered by program**)
2. Iterates each class's `classPromotions`, filtering in-memory by `$promotionProgram->getId() === $programId`
3. Groups items into `$programCampusGroups` using key `programId_campusId`
4. Deduplicates module items using key `deliveryMode_promotionPeriodId_campusId`

The bug: a single class can have multiple `classPromotion` records linked to **different promotions** (e.g., J25 and D25 cohorts) that map to **different `promotionPeriod` entities** with the **same logical period number** (e.g., both have period "1"). Since the deduplication key uses the `promotionPeriodId` (entity ID, e.g., 10 vs 20), each produces a separate calendar item — both labeled "Module 1: IN_PERSON FBL" — creating visible duplicates.

### Current Code Flow

```
findCalendarClassesByProgram($programId)
  → Returns ALL CORE classes (no program filter in SQL)

getCalendar($programId)
  → foreach $classes → foreach $classPromotions
  → filters: $promotionProgram->getId() !== $programId
  → $itemKey = $modeValue . '_' . $promotionPeriodId . '_' . $campusId
  → This key uses entity ID, not logical period number → duplicates
```

## Goals / Non-Goals

**Goals:**
- Eliminate duplicate module entries in the PM Calendar View
- Use the logical period number for deduplication instead of promotion period entity ID
- Optimize the repository query to filter by program at the database level
- Preserve the existing API response structure (`FlexSwitchCalendarDto`)

**Non-Goals:**
- Changing the calendar UI/frontend components
- Modifying the student-side calendar (different service method)
- Adding new fields to the calendar response
- Changing how campaigns are displayed in the calendar

## Decisions

### 1. Change deduplication key from entity ID to period number

**Current**: `$itemKey = $modeValue . '_' . $promotionPeriodId . '_' . $campusId`
**New**: `$itemKey = $modeValue . '_' . $period . '_' . $campusId`

The `$period` value is the logical period number (e.g., "1", "2", "3") obtained from `$promotionPeriod->getPeriod()`. This ensures that multiple `classPromotions` with the same period number are correctly collapsed into a single calendar item.

When items share the same key, the service already merges date ranges (min start_date, max end_date), so this change naturally preserves date range accuracy.

### 2. Add program filter to repository query

**File**: `bidding-api/src/Repository/ClassesRepository.php`
**Method**: `findCalendarClassesByProgram()`

Add a join path: `classes → classPromotions → promotion → program` and filter `WHERE program.id = :programId`. This avoids loading all CORE classes globally and then filtering in-memory.

### 3. Keep the `$promotionPeriodId` in the calendar item DTO

The `FlexSwitchCalendarItemDto` still carries `promotion_period_id`. When multiple `classPromotions` are collapsed, we'll keep the **first encountered** `promotion_period_id` for that logical period. This value is used by the frontend to fetch module detail via `CourseFlexByModule`, and any valid `promotionPeriodId` for the same period should return the same set of courses.

## Risks / Trade-offs

- **Minor risk**: If two promotion periods with the same period number have genuinely different course sets, collapsing them could hide data. However, the current behavior already labels them identically ("Module 1: IN_PERSON FBL") and represents them as a single logical module, so collapsing is the correct behavior.
- **Query optimization**: Adding a join to filter by program in the repository may slightly change query performance. Given the existing query already joins `course` and `courseType`, one additional join through `classPromotions → promotion → program` is trivial.
- **No backward compatibility risk**: The API response shape is unchanged. Frontend code does not need modification.
