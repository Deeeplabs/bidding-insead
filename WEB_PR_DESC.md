# Fix: Student Time Value Display Incorrectly

## Problem
Jira: [DPBFAD-919](https://insead.atlassian.net/browse/DPBFAD-919)

Students in the SGT (GMT+8) timezone were seeing bidding round closing times shifted by exactly 8 hours. Additionally, during DST transitions (e.g., March 29 for Europe/Paris), a 1-hour shift was observed between the PM configuration and student dashboard views, presenting timers that displayed "1d 23h" for mathematically configured 48-hour periods.

This issue was caused by a combination of the backend drifting timezone offsets across DST boundaries, and a **Double-Parsing Bug in the Frontend**: Components stripped timezone info during intermediate formatting into local strings, then re-parsed these local strings back as UTC using `parseUtc()`, leading to redundant offsets being applied at the final rendering step.

1. **Enrolled Courses Not Disabled in Dropdown**: In the Add/Drop & Waitlist phase, students were able to select courses they had already enrolled in from the dropdown. The system only blocked them after submission/save with a hard error ("Already enrolled in [Course]"). Enrolled courses should be preemptively disabled at the UI level.
2. **Inconsistent Validation**: Validation logic was scattered and sometimes bypassed the full course context, relying only on credit numbers.
3. **Type Inconsistency**: A mismatch in the `use-bidding-form.ts` hook led to potential runtime or build issues when passing course data to validators.
4. **Incorrect Priority of Enrolled Check**: The `is_enrolled` check in `getUnavailableReason()` was evaluated after credit/deadline/conflict checks, meaning enrolled courses could show misleading reasons instead of "Previously Enrolled".

### 1. Frontend (bidding-web)
- **`src/features/bidding/components/shared/bidding-card/CollapsePanelExtra.tsx`**: Refactored `parsedStartDate` and `parsedEndDate` to perpetually remain as `Date` objects returned by the initial `parseUtc()`, never getting downgraded to naive strings.
- **`src/features/bidding/components/bid-submission/HeaderSection.tsx`**: Applied similar refactoring to remove intermediate local-string formatting that stripped timezone context before passing to phase utilities.

## Impact
- **Accurate Time Display**: Students now see the correct local times for all bidding round transitions, matching the 8-hour gap perfectly.
- **Zero-Shift Logic**: The `Date` objects are maintained perfectly throughout the pipeline, allowing the countdown timer to mathematically strictly measure the duration gap, completely mitigating the "1d 23h" discrepancy when combined with the backend `DateHelper` fix.
- **Consistent Round Status**: Phase status calculations (Upcoming, Ongoing, Closed) and countdowns are now absolutely synchronized with the true system deadlines.

## Testing / Verification Steps
1. Create a bidding round crossing a DST boundary (e.g., March 28 00:00 to March 30 00:00).
2. Login as a Student.
3. Verify that the countdown timers display exactly "48h / 2d", resolving the "1d 23h" gap.
4. Login as a student with SGT profile to verify offset accuracy.
