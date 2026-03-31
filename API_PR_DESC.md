# Feat: Filter Dashboard Calendar by Programme and Campus

## Problem
Jira: https://insead.atlassian.net/browse/DPBFAD-927

Students could previously see all GEMBA modules in their dashboard calendar view, including those belonging to unrelated campuses (e.g., a GEMBA SGP student seeing GEMBA FNJ or GEMBA ABS modules). This cluttered the interface and caused confusion about upcoming schedules.

## Goal
Apply strict backend filtering to the `/v2/api/student/flex-switch/calendar` endpoint so it returns only modules relevant to the student's programme and active campus — with no frontend changes required. The Switch Request dropdown and all other endpoints remain unaffected.

## Changes Made

### `src/Service/FlexSwitch/FlexSwitchService.php`
- Updated `getCalendarView()` to collect the requesting student's allowed campus IDs: their home campus (`Student::getCampus()`) plus any campuses assigned via `Exchange` records (`p3Campus`, `p4Campus`, `p5Campus`).
- Extended the existing group filter to also reject groups whose `campus_id` is not in the student's allowed set, alongside the existing FlexSwitch configuration check.
- If a student has no campus assigned, the campus filter is skipped as a safe fallback (all configured-promotion groups pass through).

### `tests/Unit/Domain/FlexSwitch/FlexSwitchCalendarFilterTest.php` *(new)*
- 5 PHPUnit tests covering:
  - Home campus match included
  - Non-matching campus excluded
  - Exchange campus match included
  - Promotion without FlexSwitch config excluded regardless of campus
  - No campus filter applied when student has no campus assigned

## Impact
- **bidding-api**: `getCalendarView()` in `FlexSwitchService` now filters by both configured promotion and student campus. No controller changes.
- **bidding-web**: Zero frontend changes required — the frontend consumes the filtered payload natively.
- **Switch Request**: The `/student/flex-switch/courses` and `/student/flex-switch/switch-to-courses` endpoints use independent service methods and are unaffected.
- Safe for in-flight active campaigns.
