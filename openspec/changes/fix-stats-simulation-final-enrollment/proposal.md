## Why

The "Simulation" and "Final Enrollment" phases in the campaign detail view are displaying inaccurate statistics for student counts and course counts. This directly impacts Programme Managers' ability to make informed decisions during the bidding lifecycle.

Three distinct issues were identified:

1. **Final Enrollment student count is wrong** — it shows a smaller number than Bidding Round because it counts only students with SELECTED/ENROLLED bid status, instead of all eligible students. The expectation is that the student number should be the same from Bidding Round all the way to Final Enrollment, regardless of whether students submit bids or do add/drop.

2. **Simulation shows 0 courses immediately after campaign creation** — the `getCoursesWithEnrollmentData()` method returns early with `total: 0` when no bids exist, even though available courses are already loaded. It also fails to recalculate totals after applying a search filter.

3. **Student course queries return bids from wrong phases** — the `getStudentCourses()` and `getStudentEnrolledCourses()` methods don't filter by `moduleType`, so they may include bids from bidding_round or add_drop phases when viewing Final Enrollment.

## What Changes

- **Fix** the Final Enrollment student count to use `getEligibleStudents()` — same as Bidding Round, Simulation, and Add/Drop
- **Fix** Simulation course display to show available courses when no bids exist yet
- **Fix** Simulation search filter to recalculate `totalCourses` and `totalSections` after filtering
- **Fix** student course queries in Final Enrollment to filter by `moduleType = 'final_enrollment'`

## Capabilities

### New Capabilities
None — this is a bug fix to existing functionality.

### Modified Capabilities
- `final-enrollment-stats`: Student count must use `getEligibleStudents()` instead of `countEnrolledStudentsByCampaign()`; student course queries must filter by `moduleType`
- `simulation-stats`: Course/section counts must be calculated before the empty-bids check; search filter must recalculate totals

## Impact

### Affected Components

**bidding-api (Backend)**
- `src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php`
  - `buildFinalEnrollmentDetail()` — wrong student counting method
  - `getStudentCourses()` — missing moduleType filter
  - `getStudentEnrolledCourses()` — missing moduleType filter
- `src/Domain/Simulation/Service/SimulationDashboardService.php`
  - `getCoursesWithEnrollmentData()` — early return bug, search filter recalculation bug

### Root Cause Analysis

**1. Final Enrollment Student Count**

| Phase | Method Used | Result |
|---|---|---|
| Pre-Bidding | `getEligibleStudents()` | ✅ All eligible students |
| Bidding Round | `getEligibleStudents()` | ✅ All eligible students |
| Simulation | `getEligibleStudents()` | ✅ All eligible students |
| Add/Drop | `getEligibleStudents()` | ✅ All eligible students |
| Final Enrollment | `countEnrolledStudentsByCampaign()` | ❌ Only SELECTED/ENROLLED students |

**2. Simulation Course Count**

`getCoursesWithEnrollmentData()` checked `if (empty($courseIds))` and returned early with 0, BEFORE calculating `totalCourses` and `totalSections` from `$availableCourses`. After search filtering, `$totalCourses` was recalculated from the pagination slice instead of the full filtered set.

**3. Student Course Queries**

`getStudentCourses()` and `getStudentEnrolledCourses()` query bids by `campaign_id` and `student_id` but don't filter by `moduleType`, so they include bids from bidding_round, add_drop, etc.

### Risk Assessment
- **Low risk**: Data display bug fixes, no behavioral changes
- **No migration required**: No database schema changes
- **No API contract changes**: Same response fields, just corrected values
- **Backward compatible**: Values will now be accurate and consistent
