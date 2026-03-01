## Context

The INSEAD Bidding System's campaign detail view displays statistics for the "Simulation" and "Final Enrollment" phases. Programme Managers rely on these statistics (student count, course count) to monitor enrollment and make decisions. 

**REVISED AFTER TESTING:** After initial fixes were applied, testing revealed that Simulation and Final Enrollment stats still do not match Bidding Round phase stats. This requires a deeper analysis and revised approach.

The issue manifests in:
- **bidding-api**: `AdminCampaignDetailService::buildSimulationDetail()` - delegates to `SimulationDashboardService` for simulation statistics
- **bidding-api**: `AdminCampaignDetailService::buildFinalEnrollmentDetail()` - uses `BidRepository::countEnrolledStudentsByCampaign()` for final enrollment statistics

## Goals / Non-Goals

**Goals:**
- Fix Final Enrollment student count to filter by specific `campaignModule` (not all modules in campaign)
- Fix Simulation student count to use consistent eligible student counting logic as Bidding Round
- Ensure course counts are consistent across all phases using the same filtering logic

**Non-Goals:**
- No changes to the bidding workflow or simulation algorithm
- No database schema modifications
- No changes to the Simulation allocation engine behavior

## Decisions

### Root Cause Analysis (Post-Testing)

1. **Final Enrollment Stats Root Cause**:
   - `BidRepository::countEnrolledStudentsByCampaign($campaignId)` only filters by `campaign_id`
   - It does NOT filter by `campaignModule`, so it counts ALL students with SELECTED/ENROLLED status across ALL modules in the campaign
   - **Fix**: Add `campaignModule` parameter to filter only students enrolled in the specific Final Enrollment phase

2. **Simulation Stats Root Cause**:
   - `SimulationDashboardService::getBiddingStudentsCount()` counts from simulation table when run exists
   - When no run exists, it falls back to `BidRepository::countDistinctStudentsByCampaign()` which counts students with `submissionStatus = 'final'`
   - This may not match the eligible students count from Bidding Round
   - **Fix**: Ensure fallback uses same eligible student counting logic (or accept that it shows different numbers before simulation runs)

### Technical Approach

- **Final Enrollment Fix**: Modify `countEnrolledStudentsByCampaign()` to accept optional `$campaignModuleId` parameter
- **Simulation Fix**: Review fallback logic in `getBiddingStudentsCount()` for consistency
- **Course Count**: Ensure all phases use `campaignCourseService::getFilteredCourseData()` consistently

## Risks / Trade-offs

- **Risk**: Adding campaignModule filter may reveal edge cases where students have multiple enrollments
  - **Mitigation**: Document what counts should include (enrolled only vs. all bidders)

- **Risk**: Course filtering logic may differ slightly between phases
  - **Mitigation**: Ensure all phases use the same `getFilteredCourseData()` method

- **Risk**: Statistics may appear different before simulation runs
  - **Mitigation**: Document that Simulation shows "eligible students with bids" while Bidding Round shows "all eligible students"

## Open Questions

1. Should Final Enrollment student count include waitlisted students or only successfully enrolled students?
2. Should course count include all classes or only classes with at least one enrolled student?
3. For Simulation phase before a run is executed, what should the student count show? (eligible students vs. students with bids)
