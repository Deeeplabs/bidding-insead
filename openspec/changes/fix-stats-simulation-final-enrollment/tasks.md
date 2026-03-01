## 1. Investigation

- [x] 1.1 Analyze frontend data flow: Review `bidding-admin/src/src/campaign-management/use-detail-phase.ts` to trace `useSimulationPhase` and `useFinalEnrollmentPhase` hooks
- [x] 1.2 Identify API endpoints: Find the backend API endpoints that provide statistics data for Simulation and Final Enrollment phases
- [x] 1.3 Review backend query logic: Examine the domain services in `bidding-api/src/Domain/` responsible for calculating simulation and enrollment statistics
- [x] 1.4 Identify root cause: Determine whether the issue is in frontend display logic, API response data, or backend query calculation

## 2. Simulation Statistics Fix

- [x] 2.1 Fix the identified issue in either frontend hooks or backend domain service for Simulation statistics - **FIXED**: Fixed early return bug in SimulationDashboardService::getCoursesWithEnrollmentData that returned 0 courses when no bids existed
- [x] 2.2 If backend fix: Update the relevant domain service query to calculate correct student and course counts - **FIXED**: Changed logic to show available courses even when no bids exist yet, properly calculates totalCourses and totalSections before checking if courseIds is empty
- [x] 2.3 If frontend fix: Update the React component or hook logic to correctly transform API response - **N/A**: No frontend fix needed, issue was in backend (SimulationDashboardService)
- [ ] 2.4 Verify the fix by testing with a campaign that has known bid counts

## 3. Final Enrollment Statistics Fix

- [x] 3.1 Fix the identified issue in either frontend hooks or backend domain service for Final Enrollment statistics
- [x] 3.2 Backend fix: Update the relevant domain service query to calculate correct student and course counts - **FIXED**: Changed from counting eligible students to counting enrolled students (status SELECTED or ENROLLED)
- [x] 3.3 No frontend fix required - issue was in backend logic
- [ ] 3.4 Verify the fix by testing with a campaign that has completed enrollment

## 4. Integration Testing

- [ ] 4.1 Test Simulation statistics after creating a new campaign
- [ ] 4.2 Test Simulation statistics after editing an existing campaign
- [ ] 4.3 Test Final Enrollment statistics after campaign progresses to Final Enrollment phase
- [ ] 4.4 Test Final Enrollment statistics after editing a campaign in Final Enrollment phase
- [ ] 4.5 Verify no regression in other campaign phases (Bidding Round, Add/Drop)
