## Why

The "Simulation" and "Final Enrollment" phases in the campaign detail view are displaying inaccurate statistics for student counts and course counts. This directly impacts Programme Managers' ability to make informed decisions during the bidding lifecycle.

Four distinct issues were identified:

1. **Final Enrollment student count is wrong** — it shows a smaller number than Bidding Round because it counts only students with SELECTED/ENROLLED bid status, instead of all eligible students. The expectation is that the student number should be the same from Bidding Round all the way to Final Enrollment, regardless of whether students submit bids or do add/drop.

2. **Simulation shows 0 courses immediately after campaign creation** — the `getCoursesWithEnrollmentData()` method returns early with `total: 0` when no bids exist, even though available courses are already loaded.

3. **Student course queries return bids from wrong phases** — the `getStudentCourses()` and `getStudentEnrolledCourses()` methods don't filter by `moduleType`, so they may include bids from bidding_round or add_drop phases when viewing Final Enrollment.

4. **Simulation stats (Courses/Sections) change when search filter is applied** — the `total_sections` returned from `getCoursesWithEnrollmentData()` is recalculated after the search filter, causing the header statistics to change from e.g. "232 (3142)" to "232 (66)" when typing a search term. The stats header should remain static regardless of any search filter applied.

## What Changes

- **Fix** the Final Enrollment student count to use `getEligibleStudents()` — same as Bidding Round, Simulation, and Add/Drop
- **Fix** Simulation course display to show available courses when no bids exist yet
- **Fix** student course queries in Final Enrollment to filter by `moduleType = 'final_enrollment'`
- **Fix** Simulation stats to be static — `total_courses` and `total_sections` in the statistics block must NOT change when a search filter is applied. The search filter should only affect the course list and pagination, not the header statistics.

## Capabilities

### New Capabilities
None — this is a bug fix to existing functionality.

### Modified Capabilities
- `final-enrollment-stats`: Student count must use `getEligibleStudents()` instead of `countEnrolledStudentsByCampaign()`; student course queries must filter by `moduleType`
- `simulation-stats`: Course/section counts in statistics must be static (unaffected by search filter); search filter must only affect course list pagination

## Impact

### Affected Components

**bidding-api (Backend)**
- `src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php`
  - `buildFinalEnrollmentDetail()` — wrong student counting method
  - `getStudentCourses()` — missing moduleType filter
  - `getStudentEnrolledCourses()` — missing moduleType filter
- `src/Domain/Simulation/Service/SimulationDashboardService.php`
  - `getCoursesWithEnrollmentData()` — `total_sections` recalculated after search filter instead of remaining static
  - `getDashboardData()` — statistics block uses search-affected `total_sections`

### Root Cause Analysis

**1. Final Enrollment Student Count**

| Phase | Method Used | Result |
|---|---|---|
| Pre-Bidding | `getEligibleStudents()` | ✅ All eligible students |
| Bidding Round | `getEligibleStudents()` | ✅ All eligible students |
| Simulation | `getEligibleStudents()` | ✅ All eligible students |
| Add/Drop | `getEligibleStudents()` | ✅ All eligible students |
| Final Enrollment | `countEnrolledStudentsByCampaign()` | ❌ Only SELECTED/ENROLLED students |

**2. Simulation Stats Affected by Search Filter**

In `getCoursesWithEnrollmentData()`:
- `$totalCoursesBeforeSearch` is saved at line 473 BEFORE search → returned as `total` → ✅ static
- `$totalSections` is recalculated AFTER search filter (lines 482-495) → returned as `total_sections` → ❌ changes with filter

The `getDashboardData()` method feeds these into the `statistics` block:
- `total_courses` ← `$coursesResult['total']` (static) ✅
- `total_sections` ← `$coursesResult['total_sections']` (changes with filter) ❌

**3. Student Course Queries**

`getStudentCourses()` and `getStudentEnrolledCourses()` query bids by `campaign_id` and `student_id` but don't filter by `moduleType`, so they include bids from bidding_round, add_drop, etc.

### Risk Assessment
- **Low risk**: Data display bug fixes, no behavioral changes
- **No migration required**: No database schema changes
- **No API contract changes**: Same response fields, just corrected values
- **Backward compatible**: Values will now be accurate and consistent
