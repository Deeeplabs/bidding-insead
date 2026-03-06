## 1. Repository Query Optimization

- [x] 1.1 **Add program filter to `ClassesRepository::findCalendarClassesByProgram()`**
  - **File**: `bidding-api/src/Repository/ClassesRepository.php`
  - **Method**: `findCalendarClassesByProgram(int $programId)`
  - Add joins: `leftJoin('cl.classPromotions', 'cp')` → `leftJoin('cp.promotion', 'p')` → `leftJoin('p.program', 'prog')`
  - Add WHERE clause: `->andWhere('prog.id = :programId')->setParameter('programId', $programId)`
  - Use `SELECT DISTINCT cl, course, campus` to avoid duplicate class rows from multiple classPromotions
  - **Verify**: Run the query and confirm it returns only classes linked to the specified program

## 2. Fix Calendar Module Deduplication

- [x] 2.1 **Change deduplication key in `PM\FlexSwitchService::getCalendar()`**
  - **File**: `bidding-api/src/Service/FlexSwitch/PM/FlexSwitchService.php`
  - **Method**: `getCalendar(int $programId)` (lines 86–319)
  - **Current** (line 147): `$itemKey = $modeValue . '_' . $promotionPeriodId . '_' . $campusId;`
  - **Change to**: `$itemKey = $modeValue . '_' . $period . '_' . $campusId;`
  - The `$period` variable is already available from line 145: `$period = $promotionPeriod->getPeriod();`
  - This ensures modules with the same delivery mode, period number, and campus are collapsed into one item
  - Date range merging (lines 160–174) already handles min start_date / max end_date correctly when items share a key

## 3. Manual Verification

- [ ] 3.1 **Verify via PM admin panel**
  - Log in as PM
  - Navigate to **Switch → Dashboard → Calendar View**
  - Confirm no duplicate module rows appear for the same programme
  - Confirm each module shows correct date ranges spanning all underlying classes
  - Click on a module to verify the course list drawer shows the correct courses

- [ ] 3.2 **Verify API response directly**
  - Call `GET /api/flex-switch/calendar?program_id={id}`
  - Check that no two items within the same group have identical `module` labels
  - Confirm `start_date` and `end_date` values are correct (minimum start, maximum end across merged classes)
  - Confirm response structure matches `FlexSwitchCalendarDto` schema (no new/removed fields)
