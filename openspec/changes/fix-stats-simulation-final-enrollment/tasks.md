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

## 2. Fix Simulation Stats to Be Static (Not Affected by Search Filter)

- [x] 2.1 In `SimulationDashboardService::getCoursesWithEnrollmentData()`, save `$totalSectionsBeforeSearch` before search filtering
  - **File**: `bidding-api/src/Domain/Simulation/Service/SimulationDashboardService.php`
  - At line 473, alongside `$totalCoursesBeforeSearch = count($availableCourses)`, add:
    ```php
    $totalSectionsBeforeSearch = $totalSections;
    ```
  - This captures the total section count BEFORE the search filter is applied

- [x] 2.2 Return `$totalSectionsBeforeSearch` instead of `$totalSections` in the response
  - **File**: `bidding-api/src/Domain/Simulation/Service/SimulationDashboardService.php`
  - In the return statement (~line 641), change:
    ```php
    'total_sections' => $totalSections
    ```
  - To:
    ```php
    'total_sections' => $totalSectionsBeforeSearch
    ```
  - This ensures the statistics header shows a static section count unaffected by search

- [x] 2.3 Verify `getDashboardData()` statistics block uses static values
  - **File**: `bidding-api/src/Domain/Simulation/Service/SimulationDashboardService.php`
  - Confirm at lines 104-109 that `statistics` reads from `$coursesResult['total']` (courses) and `$coursesResult['total_sections']` (sections) — both now static
  - `total_students` already uses `getEligibleStudents()` directly — also static ✅

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
- [x] 4.3 Code-verify `total_sections` in response uses `$totalSectionsBeforeSearch` (static value)
- [x] 4.4 Code-verify student course queries include moduleType filter
- [ ] 4.5 Manual test: Verify Simulation stats header does NOT change when typing a search term
- [ ] 4.6 Manual test: Verify course list and pagination DO change when typing a search term
- [ ] 4.7 Manual test: Verify all phases show same student count
