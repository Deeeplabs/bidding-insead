## ADDED Requirements

### Requirement: Campaign Dashboard Visibility for Manual Assignees
A student who is manually added to a specific phase of a campaign (e.g., Pre-bidding ONLY) must be able to see the campaign on their dashboard and bidding listing pages, even if they do not belong to the campaign's promotion targeting.

#### Scenario: Student manually added for Pre-bidding ONLY
- **WHEN** the student is manually added to the pre-bidding phase but not normally a participant in the campaign's overall promotion rules
- **THEN** the student's dashboard will correctly list this campaign under available or upcoming campaigns.

### Requirement: Restricted Phase Visibility for Manual Round Assignments
When a student is manually assigned to participate in a specific phase (e.g., Pre-bidding ONLY), they must not see or have access to any other phases like Add/Drop or Waitlist.

#### Scenario: Student only added for Pre-bidding round
- **WHEN** the student is impersonated or logging into the system, and viewing the campaign they were manually added to for Pre-bidding ONLY
- **THEN** the bidding phase display resolves to show only the Pre-bidding step, hiding or blocking Add/Drop and Waitlist phases.

### Requirement: Manual Addition Configuration Constraint
In the Admin campaign configuration pages, manual student additions should be strictly restricted to the pre-bidding configuration.

#### Scenario: Admin attempting to manually add students
- **WHEN** configuring phases for a new or existing campaign
- **THEN** the option to manually add students is only available under the pre-bidding phase settings, prohibiting arbitrary manual additions to Add/Drop or Waitlist phases.
