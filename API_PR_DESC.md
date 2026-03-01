# Fix Simulation and Final Enrollment Statistics Display

## Problem

The "Simulation" and "Final Enrollment" phases in the campaign detail bidding view were displaying inaccurate counts for the number of students and courses. This data inconsistency occurred after creating or editing a campaign and directly impacted Programme Managers' ability to make informed decisions during the bidding lifecycle.

### Root Cause

1. **Simulation Statistics Bug:** The `SimulationDashboardService::getCoursesWithEnrollmentData` method was returning early with `total: 0` when no bids existed, even though available courses were already loaded. This caused the Simulation phase to show 0 courses and 0 sections immediately after campaign creation.

2. **Final Enrollment Statistics Bug:** The backend was counting eligible students instead of enrolled students (status SELECTED or ENROLLED), resulting in inflated enrollment numbers.

## Solution

### Changes Made

**File 1: `src/Domain/Simulation/Service/SimulationDashboardService.php`**
- Fixed the `getCoursesWithEnrollmentData` method to calculate `totalCourses` and `totalSections` from `$availableCourses` BEFORE checking if `$courseIds` is empty
- When no bids exist but available courses exist, the method now continues to show all available courses instead of returning early with 0
- Added proper handling for search filter - recalculates totals after applying search filter
- Added logging for better debugging when no courses are found

**File 2: `src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php`**
- Changed the Final Enrollment statistics query logic to count enrolled students (status SELECTED or ENROLLED) instead of eligible students
- This ensures the student count reflects actual enrollments rather than just eligibility

## Files Changed

| File | Change |
|------|--------|
| `src/Domain/Simulation/Service/SimulationDashboardService.php` | Fixed early return bug to show available courses when no bids exist |
| `src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php` | Fixed to count enrolled students (SELECTED/ENROLLED) instead of eligible students |

## Impact

* **User Experience:** Programme Managers can now see accurate course and student counts immediately after creating a campaign, during both Simulation and Final Enrollment phases
* **Data Accuracy:** Statistics now correctly reflect available courses regardless of bid activity, and enrollment counts reflect actual enrollments
* **No Breaking Changes:** The fixes maintain backward compatibility with existing API contracts

## Testing

Verified through code review and implementation:
- [x] Fixed early return logic in SimulationDashboardService to show available courses when no bids exist
- [x] Verified totalCourses and totalSections are calculated before empty check
- [x] Confirmed search filter properly recalculates totals
- [x] Fixed AdminCampaignDetailService to count enrolled students (SELECTED/ENROLLED status)
- [ ] Manual testing required: Create new campaign and verify Simulation shows correct course count
- [ ] Manual testing required: Verify Final Enrollment shows correct enrollment counts
- [ ] Manual testing required: Test after campaign editing to ensure statistics update correctly
