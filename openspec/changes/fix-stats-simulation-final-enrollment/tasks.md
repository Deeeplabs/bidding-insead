## 1. Fix Final Enrollment Student Count

- [x] 1.1 In `AdminCampaignDetailService::buildFinalEnrollmentDetail()` (~line 1232-1235), replace the `countEnrolledStudentsByCampaign()` call with `getEligibleStudents()`:
  - **File**: `bidding-api/src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php`
  - **Replace**:
    ```php
    $totalStudents = $this->bidRepository->countEnrolledStudentsByCampaign($campaign->getId(), $campaignModuleId);
    ```
  - **With**:
    ```php
    $eligibleStudents = $this->campaignStudentEligibilityService->getEligibleStudents(
        $campaign,
        $biddingConfig,
        $biddingPhaseConfigId
    );
    $totalStudents = count($eligibleStudents);
    ```
  - **Note**: `$biddingConfig` and `$biddingPhaseConfigId` are already computed at lines 1227-1229.

## 2. Fix Simulation Course Count

- [x] 2.1 In `SimulationDashboardService::getCoursesWithEnrollmentData()`, move `totalCourses` and `totalSections` calculation BEFORE the empty `$courseIds` check
  - **File**: `bidding-api/src/Domain/Simulation/Service/SimulationDashboardService.php`
  - Calculate `totalCourses` from `count($availableCourses)` and `totalSections` by iterating class promotions BEFORE checking `if (empty($courseIds))`
  - Change early return: only return 0 when both `$courseIds` is empty AND `$totalCourses` is 0
  - When no bids but courses exist, log info and continue to show all available courses
  - Add `total_sections: 0` to empty return response for consistency

- [x] 2.2 Fix search filter recalculation in `getCoursesWithEnrollmentData()`
  - After applying search filter on `$availableCourses`, recalculate `$totalCourses` and `$totalSections` from the filtered set
  - Remove the post-pagination `$totalCourses = count($availableCourses)` line (it was too late)

## 3. Fix moduleType Filters in Final Enrollment Queries

- [x] 3.1 Add `moduleType = 'final_enrollment'` filter to `getStudentCourses()` query
  - **File**: `bidding-api/src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php`
  - Add `->andWhere('b.moduleType = :moduleType')` and `->setParameter('moduleType', 'final_enrollment')` to the bid query

- [x] 3.2 Add `moduleType = 'final_enrollment'` filter to `getStudentEnrolledCourses()` query
  - **File**: `bidding-api/src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php`
  - Add `->andWhere('b.moduleType = :moduleType')` and `->setParameter('moduleType', 'final_enrollment')` to the enrolled courses query

## 4. Verification

- [x] 4.1 Code-verify all 5 phase builders use `getEligibleStudents()` for student count
- [x] 4.2 Code-verify SimulationDashboardService calculates totals before empty check
- [x] 4.3 Code-verify search filter recalculates totals after filtering
- [x] 4.4 Code-verify student course queries include moduleType filter
- [ ] 4.5 Manual test: Create campaign and verify all phases show same student count
- [ ] 4.6 Manual test: Verify Simulation shows courses immediately after campaign creation
- [ ] 4.7 Manual test: Compare stats across all phases after students submit bids and do add/drop
