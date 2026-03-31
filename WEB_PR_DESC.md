# Fix: Student Time Value Display Incorrectly

## Problem
Jira: [DPBFAD-919](https://insead.atlassian.net/browse/DPBFAD-919)

Students in the SGT (GMT+8) timezone were seeing bidding round closing times shifted by exactly 8 hours. Additionally, during DST transitions (e.g., March 29 for Fontainebleau campus), a 1-hour shift was observed between the PM configuration and student dashboard views.

This issue was caused by:
1. **Backend "Fake UTC" Issue**: `DateHelper::toIso` in `bidding-api` was formatting dates as ISO string with a `Z` suffix but *without* converting them to UTC first. This meant local server times (e.g., Paris or SGT) were being incorrectly presented as UTC to the frontend.
2. **Double-Parsing Bug (Frontend)**: Components stripped timezone info during intermediate formatting, then re-parsed these local strings as UTC using `parseUtc`, leading to redundant offsets being applied at the final rendering step.

1. **Enrolled Courses Not Disabled in Dropdown**: In the Add/Drop & Waitlist phase, students were able to select courses they had already enrolled in from the dropdown. The system only blocked them after submission/save with a hard error ("Already enrolled in [Course]"). Enrolled courses should be preemptively disabled at the UI level.
2. **Inconsistent Validation**: Validation logic was scattered and sometimes bypassed the full course context, relying only on credit numbers.
3. **Type Inconsistency**: A mismatch in the `use-bidding-form.ts` hook led to potential runtime or build issues when passing course data to validators.
4. **Incorrect Priority of Enrolled Check**: The `is_enrolled` check in `getUnavailableReason()` was evaluated after credit/deadline/conflict checks, meaning enrolled courses could show misleading reasons instead of "Previously Enrolled".

### 1. Backend (bidding-api)
- **`src/Helper/DateHelper.php`**: Refactored `toIso()` to always convert `DateTimeInterface` objects to UTC before applying the `Z` suffix. This ensures the API response is always truly in UTC.
- **`src/Controller/Api/Student/Campaign/StudentActiveCampaignController.php`**: Standardized date formatting by replacing manual `.format('Y-m-d H:i:s')` calls with `DateHelper::toIso()`. This ensures consistent UTC/ISO-8601 formatting across all student-facing endpoints.

### 2. Frontend (bidding-web)
- **`src/features/bidding/components/shared/bidding-card/CollapsePanelExtra.tsx`**: Refactored `parsedStartDate` and `parsedEndDate` to remain as `Date` objects returned by `parseUtc`, rather than formatted local strings.
- **`src/features/bidding/components/bid-submission/HeaderSection.tsx`**: Applied similar refactoring to remove intermediate local-string formatting that stripped timezone context.

## Impact
- **Accurate Time Display**: Students now see the correct local times for all bidding round transitions, including both the 8-hour SGT shift and the detailed 1-hour shift observed during the March 29 DST transition.
- **Zero-Shift Logic**: The system now correctly processes the UTC source of truth from the database through a robust UTC API layer to the student's local browser formatting.
- **Consistent Round Status**: Phase status calculations (Upcoming, Ongoing, Closed) and countdowns are now correctly synchronized with the real deadlines.

## Testing / Verification Steps
1. Create a bidding round with a closing time (e.g., 2:00 AM SGT on March 29).
2. Login as a Student with profile set to SGT.
3. Verify that the bidding round correctly displays `Starts on 29 Mar 2026, 02:00 GMT+8`.
4. Confirm that the header and card views match the PM configuration exactly, and that the 8-hour and 1-hour shifts are no longer present.
5. Verify that existing countdown timers and status indicators correctly transition at the expected local times.
