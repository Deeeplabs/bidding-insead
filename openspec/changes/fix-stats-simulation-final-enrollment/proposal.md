## Why

The "Simulation" and "Final Enrollment" phases in the campaign detail bidding view are displaying inaccurate counts for the number of students and courses. This data inconsistency occurs after creating or editing a campaign and directly impacts Programme Managers' ability to make informed decisions during the bidding lifecycle. **After initial fixes were applied, testing revealed that Simulation and Final Enrollment stats still do not match Bidding Round phase stats.**

## What Changes

- **Investigate and fix** the incorrect student and course count calculations in the Simulation phase detail view
- **Investigate and fix** the incorrect student and course count calculations in the Final Enrollment phase detail view
- Ensure all phase statistics use consistent counting logic that matches the Bidding Round baseline
- Verify that statistics update correctly after campaign creation and editing operations

## Capabilities

### New Capabilities
None - this is a bug fix to existing functionality.

### Modified Capabilities
- `simulation-stats`: The Simulation statistics calculation logic needs verification and fix to ensure accurate student and course counts
- `final-enrollment-stats`: The Final Enrollment statistics calculation logic needs verification and fix to ensure accurate student and course counts

## Impact

### Affected Components

**bidding-admin (Frontend)**
- `src/components/dashboard/process/simulation/simulation-process.tsx` - Displays simulation statistics
- `src/components/dashboard/process/final-enrollment/final-enrollment.tsx` - Displays final enrollment statistics
- `src/src/campaign-management/use-detail-phase.ts` - Contains `useSimulationPhase` and `useFinalEnrollmentPhase` hooks that fetch statistics data

**bidding-api (Backend)**
- Domain services responsible for calculating simulation and final enrollment statistics
- API endpoints that provide statistics data to the frontend

### Root Cause Investigation Required

**REVISED AFTER TESTING - Initial fixes were insufficient:**

1. **Final Enrollment Stats Issue**: The `countEnrolledStudentsByCampaign()` method in `BidRepository` does NOT filter by `campaignModule`. This causes it to count ALL enrolled students across ALL modules in the campaign, not just those from the Final Enrollment phase.
   - **Location**: `bidding-api/src/Repository/BidRepository.php::countEnrolledStudentsByCampaign()`
   - **Fix needed**: Add `campaignModule` filter to count only students enrolled in the specific Final Enrollment phase

2. **Simulation Stats Issue**: When no simulation run exists, `getBiddingStudentsCount()` falls back to counting students with `submissionStatus = 'final'` which may not match the eligible students count from Bidding Round.
   - **Location**: `bidding-api/src/Domain/Simulation/Service/SimulationDashboardService.php::getBiddingStudentsCount()`
   - **Fix needed**: Ensure fallback uses same eligible student counting logic as Bidding Round

3. **Course Count Consistency**: Ensure all phases use the same course filtering logic from `campaignCourseService::getFilteredCourseData()` to get consistent course counts.

### Testing Considerations
- Verify statistics display correctly after creating a new campaign
- Verify statistics display correctly after editing an existing campaign
- Test with various campaign configurations (different numbers of students, courses, modules)
- Ensure statistics are accurate across different campaign phases

### Risk Assessment
- **Low risk**: This is a data display bug fix, not a behavioral change
- **No migration required**: No database schema changes anticipated
- **Backward compatible**: No API contract changes expected, only data accuracy fixes
