## Context

The INSEAD Bidding System's campaign detail view displays statistics (`total_students`, `total_courses`, `total_sections`) for each phase. Programme Managers expect these numbers to be consistent and accurate across all phases.

Three bugs affect accuracy:
1. Final Enrollment uses `countEnrolledStudentsByCampaign()` instead of `getEligibleStudents()` for student count
2. Simulation returns 0 courses when no bids exist, even though available courses are loaded
3. Student course queries in Final Enrollment don't filter by `moduleType`, mixing data from other phases

## Goals / Non-Goals

**Goals:**
- Fix Final Enrollment `total_students` to use `getEligibleStudents()` — matching all other phases
- Fix Simulation `getCoursesWithEnrollmentData()` to show available courses when no bids exist
- Fix Simulation search filter to recalculate totals after filtering
- Fix student course queries to filter by `moduleType = 'final_enrollment'`

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

### 2. Simulation Course Count Fix

**Location**: `SimulationDashboardService::getCoursesWithEnrollmentData()`

**Problem**: `totalCourses` and `totalSections` were calculated AFTER the `if (empty($courseIds))` early return, so when no bids existed, the method returned 0 for everything.

**Fix**:
- Move `totalCourses`/`totalSections` calculation BEFORE the empty check
- Change early return logic: only return 0 when BOTH `$courseIds` is empty AND `$totalCourses` is 0
- When `$courseIds` is empty but courses exist, log info and continue showing all available courses
- Add `total_sections` key to empty return response for consistency

**Search filter fix**: After applying search filter on `$availableCourses`, recalculate `$totalCourses` and `$totalSections` from the filtered set (previously `$totalCourses` was calculated after pagination slice).

### 3. moduleType Filter Fix

**Location**: `AdminCampaignDetailService`

- `getStudentCourses()`: Add `b.moduleType = 'final_enrollment'` to the bid query
- `getStudentEnrolledCourses()`: Add `b.moduleType = 'final_enrollment'` to the enrolled courses query

This ensures only bids from the Final Enrollment phase are returned when viewing student course details.

## Risks / Trade-offs

- **Minimal risk**: All fixes are isolated to specific query methods. `getEligibleStudents()` is already proven across 4 other phases.
- **No performance concern**: `getEligibleStudents()` is already called in every other phase. SimulationDashboardService just reorders existing calculations.
- **Edge case**: If student filters differ between phases, Final Enrollment will always reflect the bidding round's student pool — this is the desired behavior.
