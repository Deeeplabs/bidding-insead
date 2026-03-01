# Fix Simulation and Final Enrollment Statistics Display

## Problem

The "Simulation" and "Final Enrollment" phases in the campaign detail bidding view were displaying inaccurate counts for the number of students and courses. This data inconsistency occurred after creating or editing a campaign and directly impacted Programme Managers' ability to make informed decisions during the bidding lifecycle.

After initial fixes were applied, testing revealed that Simulation and Final Enrollment stats still did not match Bidding Round phase stats. This required additional investigation and fixes.

### Root Cause Analysis

1. **Simulation Statistics Bug (Initial Fix):** The `SimulationDashboardService::getCoursesWithEnrollmentData` method was returning early with `total: 0` when no bids existed, even though available courses were already loaded. This caused the Simulation phase to show 0 courses and 0 sections immediately after campaign creation.

2. **Final Enrollment Statistics Bug (Initial Fix):** The backend was counting eligible students instead of enrolled students (status SELECTED or ENROLLED), resulting in inflated enrollment numbers.

3. **Final Enrollment Statistics Bug (Revised Fix - After Testing):** The `countEnrolledStudentsByCampaign()` method was NOT filtering by `campaignModule`, causing it to count ALL students with SELECTED/ENROLLED status across ALL modules in just those in the specific the campaign, not Final Enrollment phase. This caused stats to not match Bidding Round baseline.

## Solution

### Changes Made

**File 1: `src/Repository/BidRepository.php`**
- Modified `countEnrolledStudentsByCampaign()` to accept an optional `$campaignModuleId` parameter
- When `$campaignModuleId` is provided, the query now filters by that specific module
- This ensures Final Enrollment only counts students enrolled in that specific phase

**File 2: `src/Domain/Simulation/Service/SimulationDashboardService.php`**
- Fixed the `getCoursesWithEnrollmentData` method to calculate `totalCourses` and `totalSections` from `$availableCourses` BEFORE checking if `$courseIds` is empty
- When no bids exist but available courses exist, the method now continues to show all available courses instead of returning early with 0
- Added proper handling for search filter - recalculates totals after applying search filter
- Added logging for better debugging when no courses are found

**File 3: `src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php`**
- Updated `buildFinalEnrollmentDetail()` to pass `campaignModuleId` to the count method
- This ensures Final Enrollment stats filter by the specific module being viewed

## Files Changed

| File | Change |
|------|--------|
| `src/Repository/BidRepository.php` | Added optional `$campaignModuleId` parameter to filter students by specific module |
| `src/Domain/Simulation/Service/SimulationDashboardService.php` | Fixed early return bug to show available courses when no bids exist |
| `src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php` | Fixed to pass `campaignModuleId` to count enrolled students |

## Impact

* **User Experience:** Programme Managers can now see accurate course and student counts immediately after creating a campaign, during both Simulation and Final Enrollment phases
* **Data Accuracy:** Statistics now correctly reflect available courses regardless of bid activity, and enrollment counts reflect actual enrollments in the specific phase
* **Consistency:** Final Enrollment stats now match Bidding Round baseline by properly filtering by campaign module
* **No Breaking Changes:** The fixes maintain backward compatibility with existing API contracts

## Testing

Verified through code review and implementation:
- [x] Fixed early return logic in SimulationDashboardService to show available courses when no bids exist
- [x] Verified totalCourses and totalSections are calculated before empty check
- [x] Confirmed search filter properly recalculates totals
- [x] Fixed BidRepository to accept optional campaignModuleId parameter for filtering
- [x] Fixed AdminCampaignDetailService to pass campaignModuleId to count method
- [ ] Manual testing required: Create new campaign and verify Simulation shows correct course count
- [ ] Manual testing required: Verify Final Enrollment shows correct enrollment counts (should match Bidding Round)
- [ ] Manual testing required: Test after campaign editing to ensure statistics update correctly
- [ ] Manual testing required: Compare stats across all three phases (Bidding Round, Simulation, Final Enrollment) for consistency
