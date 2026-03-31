# Feat: Filter Dashboard Calendar by Programme and Campus (incl. FLEX)

## Problem
Jira: https://insead.atlassian.net/browse/DPBFAD-927

Students could previously see all GEMBA modules in their dashboard calendar view, including those belonging to unrelated campuses (e.g., a GEMBA SGP student seeing GEMBA FNJ or GEMBA ABS modules). This cluttered the interface and caused confusion about upcoming schedules. Additionally, FLEX (virtual/hybrid delivery) campus modules were being filtered out for students whose home campus was not FLEX.

## Goal
Apply strict backend filtering to the `/v2/api/student/flex-switch/calendar` endpoint so it returns only modules relevant to the student's programme and allowed campuses — including FLEX campus modules for all students — with no frontend changes required. The Switch Request dropdown and all other endpoints remain unaffected.

## Changes Made

### `src/Service/FlexSwitch/FlexSwitchService.php`
- Updated `getCalendarView()` to collect the requesting student's allowed campus IDs: their home campus (`Student::getCampus()`) plus any campuses assigned via `Exchange` records (`p3Campus`, `p4Campus`, `p5Campus`).
- **Added FLEX campus inclusion**: looks up the Campus entity with `shortName = 'FLEX'` via the EntityManager and adds its ID to the allowed campus set, so all students always see flex/virtual delivery modules.
- Extended the existing group filter to reject groups whose `campus_id` is not in the student's allowed set, alongside the existing FlexSwitch configuration check.
- If a student has no campus assigned, the campus filter is skipped as a safe fallback (all configured-promotion groups pass through).
- If no FLEX campus exists in the database, the lookup is gracefully skipped without error.

### `tests/Unit/Domain/FlexSwitch/FlexSwitchCalendarFilterTest.php` *(updated)*
- 7 PHPUnit tests covering:
  - Home campus match included
  - Non-matching campus excluded
  - Exchange campus match included
  - Promotion without FlexSwitch config excluded regardless of campus
  - No campus filter applied when student has no campus assigned
  - **FLEX campus modules always included for all students**
  - **No error when FLEX campus does not exist in database**

## Impact
- **bidding-api**: `getCalendarView()` in `FlexSwitchService` now filters by configured promotion, student campus, and always includes the FLEX campus. No controller changes, no new dependencies injected (uses existing EntityManager).
- **bidding-web**: Zero frontend changes required — the frontend consumes the filtered payload natively.
- **Switch Request**: The `/student/flex-switch/courses` and `/student/flex-switch/switch-to-courses` endpoints use independent service methods and are unaffected.
- Safe for in-flight active campaigns.
