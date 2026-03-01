# Fix Simulation and Final Enrollment Statistics Display

## Problem

The "Simulation" and "Final Enrollment" phases in the campaign detail view were displaying incorrect statistics, notably fluctuating course counts and incorrect student counts. The expectations are:
1. The `total_students` statistic should be **the same across all phases** (Bidding Round → Simulation → Add/Drop → Final Enrollment), since it represents the number of eligible students in the campaign.
2. The `total_courses` and `total_sections` in the statistics header should be **static** and not change when a user applies a search filter.

### Root Cause

Four distinct issues were identified:

**1. Final Enrollment Student Count**

In `AdminCampaignDetailService::buildFinalEnrollmentDetail()`, the student count used `countEnrolledStudentsByCampaign()` which only counted students with SELECTED/ENROLLED bid status. This resulted in a smaller number than the Bidding Round, which correctly uses `getEligibleStudents()` to count all eligible students.

**2. Simulation Stats Affected by Search Filter (Fluctuating Stats)**

In `SimulationDashboardService::getCoursesWithEnrollmentData()`, `$totalSections` was recalculated *after* the search filter was applied. Since the statistics block reads from this value, typing a search term caused the overall stats header to change (e.g., from "232 (3142)" to "232 (66)"). The stats header should remain static.

**3. Simulation Course Count on Empty Bids**

The `getCoursesWithEnrollmentData()` method was returning early with `total: 0` when no bids existed (`$courseIds` was empty), even though available courses were already loaded. This caused the Simulation phase to show 0 courses and 0 sections immediately after campaign creation.

**4. Student Course Queries Missing moduleType Filter**

The `getStudentCourses()` and `getStudentEnrolledCourses()` methods in `AdminCampaignDetailService` were not filtering by `moduleType = 'final_enrollment'`, potentially returning bids from other phases.

## Solution

### File 1: `src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php`

**Change 1 — Fix Final Enrollment student count**
- Replaced `$this->bidRepository->countEnrolledStudentsByCampaign()` with `$this->campaignStudentEligibilityService->getEligibleStudents()`
- This makes Final Enrollment use the exact same student counting logic as all other phases

**Change 2 — Add moduleType filter to student queries** 
- Added `b.moduleType = 'final_enrollment'` filter to both `getStudentCourses()` and `getStudentEnrolledCourses()`

### File 2: `src/Domain/Simulation/Service/SimulationDashboardService.php`

**Change 3 — Make Simulation Stats Static**
- Saved `$totalSectionsBeforeSearch = $totalSections` *before* applying the search filter.
- Returned `$totalSectionsBeforeSearch` as the `total_sections` in the response payload.
- This ensures the statistics header (`total`, `total_sections`) remains static and accurately reflects the full campaign catalog, while the search filter only affects the course list and pagination.

**Change 4 — Fix early return when no bids exist**
- Moved `totalCourses` and `totalSections` calculations BEFORE the empty `$courseIds` check.
- When no bids exist but available courses do, it now shows all available courses instead of returning 0.

## Impact

- **Consistent Student Counts:** All phases now show the exact same `total_students` value (eligible students).
- **Static Header Statistics:** The Simulation phase header statistics no longer fluctuate when using the search bar.
- **Accurate Empty States:** Simulation phase correctly shows available courses even before any bids are placed.
- **Data Accuracy:** Student course queries in Final Enrollment are properly scoped to the phase.
- **Backward Compatible:** No database schema changes, and no API contract changes — same response fields, just corrected runtime values.
