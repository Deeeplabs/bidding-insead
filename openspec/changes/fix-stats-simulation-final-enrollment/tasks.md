## 1. Investigation

- [x] 1.1 Analyze frontend data flow: Review `bidding-admin/src/src/campaign-management/use-detail-phase.ts` to trace `useSimulationPhase` and `useFinalEnrollmentPhase` hooks
- [x] 1.2 Identify API endpoints: Find the backend API endpoints that provide statistics data for Simulation and Final Enrollment phases
- [x] 1.3 Review backend query logic: Examine the domain services in `bidding-api/src/Domain/` responsible for calculating simulation and enrollment statistics
- [x] 1.4 Identify root cause: Determine whether the issue is in frontend display logic, API response data, or backend query calculation

## 2. Initial Fixes Applied (Insufficient - Testing Failed)

- [x] 2.1 Initial fix: Fixed early return bug in SimulationDashboardService::getCoursesWithEnrollmentData that returned 0 courses when no bids existed
- [x] 2.2 Initial fix: Changed logic to show available courses even when no bids exist yet, properly calculates totalCourses and totalSections before checking if courseIds is empty
- [x] 2.3 Initial fix: No frontend fix needed - issue was in backend (SimulationDashboardService)
- [ ] 2.4 Verify the fix by testing with a campaign that has known bid counts

## 3. Revised Fixes After Testing Failed

### 3.1 Final Enrollment Stats Fix (PRIMARY ISSUE)

- [x] 3.1.1 Fix `BidRepository::countEnrolledStudentsByCampaign()` to accept `$campaignModuleId` parameter
  - **Location**: `bidding-api/src/Repository/BidRepository.php`
  - **Issue**: Method counts ALL students with SELECTED/ENROLLED status across ALL modules in campaign
  - **Fix**: Add optional `$campaignModuleId` filter to count only students in the specific Final Enrollment phase
- [x] 3.1.2 Update `AdminCampaignDetailService::buildFinalEnrollmentDetail()` to pass `campaignModule` to the count method
  - **Location**: `bidding-api/src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php::buildFinalEnrollmentDetail()`
  - **Issue**: Currently only passes `$campaign->getId()` without campaignModule filter
- [ ] 3.1.3 Test Final Enrollment statistics after fix - verify student count matches Bidding Round baseline

### 3.2 Simulation Stats Fix (SECONDARY ISSUE)

- [x] 3.2.1 Review `SimulationDashboardService::getBiddingStudentsCount()` fallback logic
  - **Location**: `bidding-api/src/Domain/Simulation/Service/SimulationDashboardService.php::getBiddingStudentsCount()`
  - **Issue**: When no simulation run exists, falls back to counting students with `submissionStatus = 'final'`
  - **Analysis**: This is acceptable for "before simulation runs" state - document the difference
- [ ] 3.2.2 Verify Simulation student count AFTER simulation run executes (should match simulation table count)
- [ ] 3.2.3 Test Simulation statistics after fix

### 3.3 Course Count Consistency

- [x] 3.3.1 Verify all phases use `campaignCourseService::getFilteredCourseData()` for course counting
  - Bidding Round: ✅ Uses `getFilteredCourseData()`
  - Simulation: ✅ Uses `campaignCourseService::buildAvailableCourse()` (similar logic)
  - Final Enrollment: ✅ Uses `getFilteredCourseData()`
- [x] 3.3.2 Ensure course count filters are consistent (course type, promotion, class filters)
  - Analysis: All phases use similar course filtering logic

## 4. Integration Testing

- [ ] 4.1 Test Bidding Round statistics (baseline)
- [ ] 4.2 Test Simulation statistics after creating a new campaign
- [ ] 4.3 Test Simulation statistics after running simulation
- [ ] 4.4 Test Final Enrollment statistics after campaign progresses to Final Enrollment phase
- [ ] 4.5 Compare statistics across all three phases - verify consistency
- [ ] 4.6 Test Final Enrollment statistics after editing a campaign in Final Enrollment phase
- [ ] 4.7 Verify no regression in other campaign phases (Add/Drop)
