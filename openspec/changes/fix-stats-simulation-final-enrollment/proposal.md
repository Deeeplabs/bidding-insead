## Why

The "Simulation" and "Final Enrollment" phases in the campaign detail bidding view are displaying inaccurate counts for the number of students and courses. This data inconsistency occurs after creating or editing a campaign and directly impacts Programme Managers' ability to make informed decisions during the bidding lifecycle. Accurate statistics are critical for proper resource allocation and enrollment management.

## What Changes

- **Investigate and fix** the incorrect student and course count calculations in the Simulation phase detail view
- **Investigate and fix** the incorrect student and course count calculations in the Final Enrollment phase detail view
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
1. Check the API response data for Simulation and Final Enrollment statistics
2. Verify the query logic in `useSimulationPhase` and `useFinalEnrollmentPhase` hooks
3. Examine the backend domain services that calculate these statistics
4. Identify whether the issue is in data aggregation, filtering, or display logic

### Testing Considerations
- Verify statistics display correctly after creating a new campaign
- Verify statistics display correctly after editing an existing campaign
- Test with various campaign configurations (different numbers of students, courses, modules)
- Ensure statistics are accurate across different campaign phases

### Risk Assessment
- **Low risk**: This is a data display bug fix, not a behavioral change
- **No migration required**: No database schema changes anticipated
- **Backward compatible**: No API contract changes expected, only data accuracy fixes
