# Fix: Student Time Value Display Incorrectly

## Problem
Jira: [DPBFAD-919](https://insead.atlassian.net/browse/DPBFAD-919)

Students in the SGT (GMT+8) timezone were seeing bidding round closing times shifted by exactly 8 hours. For example, a round scheduled by the PM to close at 1:20 AM SGT was incorrectly displayed to students as closing at 9:20 AM SGT.

This issue was caused by a **double-parsing bug** in the frontend:
1. Components parsed the UTC date string from the API into a `Date` object.
2. The `Date` object was then formatted into a local time string *without* timezone information (e.g., `"2026-03-27 01:20:00"` for SGT).
3. This local string was passed to phase status and display utilities, which re-parsed it. Since the string had no timezone indicator, `parseUtc` assumed it was UTC and appended `Z` (`2026-03-27T01:20:00Z`).
4. Final rendering added the timezone offset a second time, resulting in the reported 8-hour shift.

## Goal
- Resolve the 8-hour shift in time display for the Student Portal.
- Ensure all bidding round opening/closing times and countdowns are accurate to the student's local timezone.

1. **Enrolled Courses Not Disabled in Dropdown**: In the Add/Drop & Waitlist phase, students were able to select courses they had already enrolled in from the dropdown. The system only blocked them after submission/save with a hard error ("Already enrolled in [Course]"). Enrolled courses should be preemptively disabled at the UI level.
2. **Inconsistent Validation**: Validation logic was scattered and sometimes bypassed the full course context, relying only on credit numbers.
3. **Type Inconsistency**: A mismatch in the `use-bidding-form.ts` hook led to potential runtime or build issues when passing course data to validators.
4. **Incorrect Priority of Enrolled Check**: The `is_enrolled` check in `getUnavailableReason()` was evaluated after credit/deadline/conflict checks, meaning enrolled courses could show misleading reasons instead of "Previously Enrolled".

### 1. `bidding-web/src/features/bidding/components/shared/bidding-card/CollapsePanelExtra.tsx`
- Refactored `parsedStartDate` and `parsedEndDate` to remain as `Date` objects returned by `parseUtc`, rather than formatted local strings.
- Updated `getPhaseStatus` and `getPhaseDisplayText` to pass these `Date` objects directly, preserving the original UTC context.
- Switched to `new Date()` for the `now` parameter in status checks to ensure comparison against the current instant.

### 2. `bidding-web/src/features/bidding/components/bid-submission/HeaderSection.tsx`
- Applied similar refactoring to remove intermediate local-string formatting that stripped timezone context.
- Updated `useMemo` dependencies to correctly track the `Date` objects and ensure status updates trigger appropriately.

## Impact
- **No API changes required** — The backend correctly provides UTC date-time values.
- **Accurate Time Display** — Students now see the correct local times for all bidding round transitions.
- **Correct Countdown Logic** — Phase status calculations (Upcoming, Ongoing, Closed) are now correctly synchronized with the real deadlines.

## Testing / Verification Steps
1. Create a bidding round with a closing time (e.g., 1:20 AM SGT).
2. Login as a Student with profile set to SGT.
3. Verify that the bidding round card in the list view shows the correct time (e.g. `Closed on 27 Mar 2026, 01:20 GMT+8`).
4. Verify that the Header Section in the bidding round page also displays matching time information and accurate countdowns.
