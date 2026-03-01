## Context

The INSEAD Bidding System's campaign detail view displays statistics for the "Simulation" and "Final Enrollment" phases. Programme Managers rely on these statistics (student count, course count) to monitor enrollment and make decisions. Currently, these counts are inaccurate after creating or editing a campaign.

The issue manifests in:
- **bidding-admin**: `src/components/dashboard/process/simulation/simulation-process.tsx` - displays Simulation statistics via `useSimulationPhase` hook
- **bidding-admin**: `src/components/dashboard/process/final-enrollment/final-enrollment.tsx` - displays Final Enrollment statistics via `useFinalEnrollmentPhase` hook

## Goals / Non-Goals

**Goals:**
- Identify the root cause of inaccurate student and course counts in Simulation and Final Enrollment statistics
- Fix the calculation logic to display accurate counts
- Ensure statistics are correctly updated after campaign creation and editing

**Non-Goals:**
- No changes to the bidding workflow or simulation algorithm
- No database schema modifications
- No changes to the Simulation allocation engine behavior

## Decisions

### Investigation Approach
1. **Frontend Data Flow Analysis**: Trace the data flow from API response to UI display in both `useSimulationPhase` and `useFinalEnrollmentPhase` hooks
2. **Backend API Investigation**: Examine the API endpoints that provide statistics data to identify where the calculation goes wrong
3. **Query Logic Review**: Check if filtering criteria are correct (e.g., correct campaign, session, module references)

### Technical Approach
- **If root cause is in frontend**: Fix the React Query hooks or component logic in `bidding-admin`
- **If root cause is in backend**: Fix the domain service query logic in `bidding-api`
- **No API contract changes**: Only fix data accuracy, preserve existing response shapes

## Risks / Trade-offs

- **Risk**: Statistics may be cached in React Query, causing stale data
  - **Mitigation**: Ensure proper cache invalidation after campaign updates

- **Risk**: Fixing the count may reveal legitimate edge cases (e.g., waitlisted students)
  - **Mitigation**: Document what counts should include (enrolled only vs. all bidders)

- **Risk**: Concurrent campaign operations may affect statistics
  - **Mitigation**: Use database transactions for consistent reads

## Open Questions

1. Should "student count" include waitlisted students or only successfully enrolled students?
2. Should "course count" include all classes or only classes with at least one enrolled student?
3. After editing a campaign, do statistics need to be manually refreshed or should they update automatically?
