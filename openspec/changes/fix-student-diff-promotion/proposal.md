## Why

Students who are manually added to a campaign for "Pre-bidding ONLY" (especially those from a different promotion) currently cannot see the campaign on their dashboard or bidding page. They are expected to see the campaign available for pre-bidding. Furthermore, manual student addition is only intended for the pre-bidding configuration. Currently, if a student is added for only a specific round (like pre-bidding), impersonating them displays all bidding phases including Add/Drop and Waitlist, which is incorrect. Students should only see and participate in the specific round they were added to.

## What Changes

- Modify the student dashboard and bidding page logic to ensure campaigns are visible to students who were manually added for "Pre-bidding ONLY," even if they belong to a different promotion.
- Constrain manual student addition strictly to the pre-bidding configuration.
- Update the student view (and impersonation view) to ensure that if a student is only added for a specific round (e.g., pre-bidding), only that specific round's step will appear. Other phases like Add/Drop and Waitlist should be hidden or disabled for these students.

## Capabilities

### New Capabilities
- `manual-round-participation`: Ensuring students only see and participate in the specific bidding rounds they were manually added to (e.g., seeing only Pre-bidding and not Add/Drop or Waitlist).

### Modified Capabilities
- `student-dashboard-visibility`: Fixing the visibility of campaigns on the student dashboard/bidding page for students manually added from a different promotion for pre-bidding only.

## Impact

- **Student Dashboard API / View:** The logic that fetches available campaigns for a student needs to be updated. It must check for manual additions to pre-bidding rounds, ignoring promotion mismatches.
- **Bidding Phase Display Logic:** The UI and backend logic that determines which bidding phases (Pre-bidding, Add/Drop, Waitlist) are visible and accessible to a student must be updated to respect the "specific round only" configuration for manually added students.
- **Admin Campaign Configuration:** Ensure that manual student addition is restricted to the pre-bidding configuration.
