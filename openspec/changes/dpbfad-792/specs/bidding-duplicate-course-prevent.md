# bidding-duplicate-course-prevent

## Overview
~~Prevent users from submitting bids for the same exact course across multiple parallel bidding rounds during the Bidding phase.~~

**REVISED**: Cross-round duplicate prevention is **removed** from the Bidding phase. Students ARE allowed to submit the same bids for the same courses in 2 or more active parallel bidding rounds. The only restrictions during the Bidding phase are:
1. Cannot bid on courses the student is **already enrolled in** from prior campaigns (`validateNoPreviousEnrollment`).
2. Cannot bid on **time-conflicting** courses (`validateTimeConflicts`).

## Requirements (Revised)

1. **Allow Cross-Round Bidding Duplicates:** During the Active Bidding phase, students can submit bids for the same course across multiple parallel bidding rounds (e.g., BIDDING1 and BIDDING2). The `validateNoParallelRoundDuplicates()` call must be removed from `BidValidator::validate()`.
2. **Keep Previously Enrolled Blocking:** `validateNoPreviousEnrollment()` remains — students cannot bid on courses they are enrolled in from prior campaigns within the same programme.
3. **Keep Time Conflict Validation:** Primary bids still validate time conflicts. Backup bids skip time conflict checks.
4. **Keep Within-Submission Duplicate Validation:** `validateNoDuplicates()` remains — students cannot submit the exact same class twice in a single submission. Same course different section is allowed if one is a backup.

## Example Flow (Revised)
- User submits a bid for Course A in `BIDDING1`.
- User navigates to `BIDDING2` (within the same program) and bids on Course A again.
- The system **allows** both bids. No validation error is thrown.
- Post-simulation, the admin system handles any resulting duplicate enrollments during the Add/Drop phase.

## Notes
- The `BidRepository::findSubmittedCourseIdsInParallelRoundsByStudentAndProgram()` query is retained for potential Add/Drop phase use, but is no longer called from `BidValidator`.
- Fix for `BidStatus::SUBMITTED` → `BidStatus::SELECTED` and `b.moduleId` → `b.campaignModule` remains applicable.
- Duplicate course resolution responsibility shifts to the Add/Drop phase via `AddDropValidator::validateNoUnresolvedDuplicateEnrollments()`.