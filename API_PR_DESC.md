# Fix Simulation and Final Enrollment Statistics Display

## Problem

The "Simulation" and "Final Enrollment" phases in the campaign detail view were displaying incorrect student counts. The expectation is that the `total_students` statistic should be **the same across all phases** (Bidding Round → Simulation → Add/Drop → Final Enrollment), since it represents the number of eligible students in the campaign — regardless of whether students submit bids, do add/drop, or not.

### Root Cause

**1. Final Enrollment Student Count (Primary Issue)**

In `AdminCampaignDetailService::buildFinalEnrollmentDetail()`, the student count was using a completely different method than all other phases:

| Phase | Method Used | What It Counted |
|---|---|---|
| Pre-Bidding | `getEligibleStudents()` | ✅ All eligible students |
| Bidding Round | `getEligibleStudents()` | ✅ All eligible students |
| Simulation | `getEligibleStudents()` | ✅ All eligible students |
| Add/Drop | `getEligibleStudents()` | ✅ All eligible students |
| **Final Enrollment** | `countEnrolledStudentsByCampaign()` | ❌ Only students with SELECTED/ENROLLED bid status |

This meant Final Enrollment showed a **smaller** number than Bidding Round because it only counted students who had actually been enrolled, not all eligible students.

**2. Simulation Course Count (Previous Fix)**

The `SimulationDashboardService::getCoursesWithEnrollmentData()` method was returning early with `total: 0` when no bids existed (`$courseIds` was empty), even though available courses were already loaded. This caused the Simulation phase to show 0 courses and 0 sections immediately after campaign creation.

**3. Student Course Queries Missing moduleType Filter**

The `getStudentCourses()` and `getStudentEnrolledCourses()` methods in `AdminCampaignDetailService` were not filtering by `moduleType = 'final_enrollment'`, which could return bids from other phases.

## Solution

### File 1: `src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php`

**Change 1 — Fix Final Enrollment student count (buildFinalEnrollmentDetail)**
- Replaced `$this->bidRepository->countEnrolledStudentsByCampaign()` with `$this->campaignStudentEligibilityService->getEligibleStudents()`
- Uses the bidding round config (`$biddingConfig`, `$biddingPhaseConfigId`) which is already computed in the method
- This makes Final Enrollment use the exact same student counting logic as all other phases

```php
// BEFORE (wrong — counted only enrolled students):
$totalStudents = $this->bidRepository->countEnrolledStudentsByCampaign($campaign->getId(), $campaignModuleId);

// AFTER (correct — counts all eligible students, same as Bidding Round):
$eligibleStudents = $this->campaignStudentEligibilityService->getEligibleStudents(
    $campaign,
    $biddingConfig,
    $biddingPhaseConfigId
);
$totalStudents = count($eligibleStudents);
```

**Change 2 — Add moduleType filter to getStudentCourses()**
- Added `b.moduleType = 'final_enrollment'` filter to the student courses query
- Ensures only final enrollment bids are returned when viewing student course details

**Change 3 — Add moduleType filter to getStudentEnrolledCourses()**
- Added `b.moduleType = 'final_enrollment'` filter to the enrolled courses query
- Ensures only final enrollment enrolled courses are returned

### File 2: `src/Domain/Simulation/Service/SimulationDashboardService.php`

**Change 1 — Fix early return when no bids exist (getCoursesWithEnrollmentData)**
- Moved `totalCourses` and `totalSections` calculation BEFORE the empty `$courseIds` check
- When no bids exist but available courses exist, the method now shows all available courses instead of returning 0
- Added `total_sections` key to the empty return response for consistency

**Change 2 — Fix search filter recalculation**
- After applying search filter on `$availableCourses`, now recalculates `$totalCourses` and `$totalSections`
- Previously, `$totalCourses` was calculated after pagination slice which gave wrong total

**Change 3 — Improved logging**
- Changed warning to info when no bids exist but courses are available (not an error condition)
- Added `available_courses` count to log message for debugging

## Files Changed

| File | Changes |
|------|---------|
| `src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php` | Fixed Final Enrollment student count to use `getEligibleStudents()`; Added `moduleType` filter to student course queries |
| `src/Domain/Simulation/Service/SimulationDashboardService.php` | Fixed early return bug; Fixed search filter recalculation; Improved logging |

## Impact

- **Student Count Consistency:** All phases (Pre-Bidding, Bidding Round, Simulation, Add/Drop, Final Enrollment) now show the same `total_students` value
- **Simulation Courses:** Simulation phase correctly shows available courses even when no bids exist yet
- **Data Accuracy:** Student course queries in Final Enrollment are now properly scoped to `moduleType = 'final_enrollment'`
- **No Breaking Changes:** No API contract changes — same response fields, just corrected values
- **No Migration Required:** No database schema changes

## Testing

- [x] Code-verified: All 5 phase builders use `getEligibleStudents()` for student count
- [x] Code-verified: SimulationDashboardService calculates totals before empty check
- [x] Code-verified: Search filter recalculates totals after filtering
- [x] Code-verified: Student course queries include moduleType filter
- [ ] Manual test: Create campaign and verify all phases show same student count
- [ ] Manual test: Verify Simulation shows courses immediately after campaign creation
- [ ] Manual test: Compare stats across all phases after students submit bids and do add/drop
