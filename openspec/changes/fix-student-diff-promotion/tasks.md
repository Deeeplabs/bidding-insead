## 1. Backend: Dashboard & Campaign Visibility

- [x] 1.1 Locate the repository or service method responsible for fetching available campaigns for a student's dashboard and bidding page. This is likely in a `CampaignRepository` or a dedicated dashboard query service.
- [x] 1.2 Modify the query to correctly include records where the student is manually assigned to the campaign (e.g., via a manual participant or assignee relationship), actively ignoring the promotion mismatch restriction if the student was manually added.
- [x] 1.3 Add a PHPUnit test verifying that a student manually added, who has a different promotion, correctly retrieves the campaign in their dashboard/bidding listing.

## 2. Backend: Manual Student Addition Constraint

- [x] 2.1 Locate the admin campaign controller or service handling the addition of manual students to a campaign.
- [x] 2.2 Add restrictive logic/validation to ensure that manual additions are only permitted during or specifically targeted to the pre-bidding configuration phase. Return an error or disable the option for other phases (Add/Drop, Waitlist).
- [x] 2.3 Write tests validating that attempting to add a manual student to any phase other than pre-bidding fails with validation errors.

## 3. Backend: Bidding Phase Visibility

- [x] 3.1 Identify the service or resolver responsible for computing and serving the visible bidding phases (Pre-bidding, Add/Drop, Waitlist) a student has access to in a campaign.
- [x] 3.2 Update the phase resolution logic: if the student is identified as a manual addition specifically allocated to only the pre-bidding round, forcefully exclude Add/Drop and Waitlist phases from the returned list of phases.
- [x] 3.3 Add a test ensuring that a manually added student impersonating their bidding view only sees the Pre-bidding phase.

## 4. Final Smoke Test

- [ ] 4.1 Perform a complete manual verification across Bidding Web (Student) and Bidding Admin platforms. Verify the UI dashboard correctly renders the manual assignments and respects phase visibility.
