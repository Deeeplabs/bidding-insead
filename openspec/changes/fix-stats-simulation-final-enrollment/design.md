## Context

The INSEAD Bidding System's campaign detail view displays statistics (`total_students`, `total_courses`, `total_sections`) for each phase. Programme Managers expect these numbers to be consistent and accurate across all phases, and to remain static regardless of search filters applied to the course list.

Four bugs affect accuracy:
1. Final Enrollment uses `countEnrolledStudentsByCampaign()` instead of `getEligibleStudents()` for student count
2. Simulation returns 0 courses when no bids exist, even though available courses are loaded
3. Student course queries in Final Enrollment don't filter by `moduleType`, mixing data from other phases
4. Simulation header stats (`total_sections`) change when search filter is applied — stats should be static

## Goals / Non-Goals

**Goals:**
- Fix Final Enrollment `total_students` to use `getEligibleStudents()` — matching all other phases
- Fix Simulation `getCoursesWithEnrollmentData()` to show available courses when no bids exist
- Fix student course queries to filter by `moduleType = 'final_enrollment'`
- Fix Simulation statistics to be static — `total_courses` and `total_sections` must NOT change when a search filter is applied

**Non-Goals:**
- No changes to bidding workflow or simulation algorithm
- No database schema modifications
- No changes to how per-course enrollment data is calculated

## Decisions

### 1. Final Enrollment Student Count Fix

**Location**: `AdminCampaignDetailService::buildFinalEnrollmentDetail()` (~line 1232-1235)

Replace:
```php
$totalStudents = $this->bidRepository->countEnrolledStudentsByCampaign($campaign->getId(), $campaignModuleId);
```

With:
```php
$eligibleStudents = $this->campaignStudentEligibilityService->getEligibleStudents(
    $campaign,
    $biddingConfig,
    $biddingPhaseConfigId
);
$totalStudents = count($eligibleStudents);
```

`$biddingConfig` and `$biddingPhaseConfigId` are already computed at lines 1227-1229. This uses the bidding round config intentionally — Final Enrollment represents the same student pool as the bidding phase.

### 2. Simulation Stats Must Be Static (Not Affected by Search Filter)

**Location**: `SimulationDashboardService::getCoursesWithEnrollmentData()`

**Problem**: `$totalSections` is recalculated after the search filter is applied (lines 482-495), so when the user types a search term, the header statistics change from e.g. "232 (3142)" to "232 (66)". Meanwhile `$totalCoursesBeforeSearch` is correctly saved before search and stays static.

**Fix**:
- Save `$totalSectionsBeforeSearch` at line 473 (alongside `$totalCoursesBeforeSearch`), capturing the total section count BEFORE any search filtering
- Return `$totalSectionsBeforeSearch` as the `total_sections` in the response (used for the statistics header)
- The search filter should only recalculate totals for pagination purposes (`$totalCourses` for `pagination.total` and `$totalPages`)
- `$totalSections` after search is only needed for pagination context, NOT for creating the statistics header

**Data flow after fix**:
```
getCoursesWithEnrollmentData() returns:
  'total'          => $totalCoursesBeforeSearch   (static, for statistics header)
  'total_sections' => $totalSectionsBeforeSearch  (static, for statistics header)
  'pagination'     => [
      'total' => $totalCourses                    (filtered, for pagination)
      ...
  ]
```

`getDashboardData()` uses:
```php
$statistics = [
    'total_courses'  => $coursesResult['total'],           // static ✅
    'total_sections' => $coursesResult['total_sections'],  // now static ✅
    'total_students' => count(getEligibleStudents(...)),   // static ✅
];
```

### 3. moduleType Filter Fix

**Location**: `AdminCampaignDetailService`

- `getStudentCourses()`: Add `b.moduleType = 'final_enrollment'` to the bid query
- `getStudentEnrolledCourses()`: Add `b.moduleType = 'final_enrollment'` to the enrolled courses query

This ensures only bids from the Final Enrollment phase are returned when viewing student course details.

## Risks / Trade-offs

- **Minimal risk**: All fixes are isolated to specific query methods. `getEligibleStudents()` is already proven across 4 other phases.
- **No performance concern**: `getEligibleStudents()` is already called in every other phase. SimulationDashboardService just saves an extra variable before search.
- **Edge case**: If student filters differ between phases, Final Enrollment will always reflect the bidding round's student pool — this is the desired behavior.
- **Search UX**: The course list and pagination will still reflect filtered results. Only the header statistics remain static, which is the expected behavior matching the existing dashboard pattern.
