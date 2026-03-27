## ADDED Requirements

### Requirement: Campaigns sorted by latest activity date
The student active-campaigns API endpoint SHALL return campaigns ordered by `COALESCE(updatedAt, createdAt) DESC`, so that newly created or recently updated campaigns appear first in the list.

#### Scenario: Newly created campaign appears at top
- **WHEN** a PM creates a new campaign that includes the student's promotion
- **THEN** the new campaign SHALL appear as the first item in the student's active-campaigns list

#### Scenario: Recently edited campaign moves to top
- **WHEN** a PM edits an existing campaign's flow (e.g., changes phase config, adds modules)
- **THEN** the campaign's `updatedAt` timestamp is updated and the campaign SHALL move to the top of the student's active-campaigns list

#### Scenario: Campaign with only createdAt (no updates)
- **WHEN** a campaign has never been updated after creation (updatedAt is null)
- **THEN** the system SHALL use `createdAt` as the sort key for that campaign

#### Scenario: Multiple campaigns with same date
- **WHEN** two campaigns have the same effective sort date
- **THEN** the system SHALL return them in a stable, deterministic order (by id DESC as tiebreaker)

#### Scenario: Sorting applies across all campaigns
- **WHEN** a student has both active and completed campaigns
- **THEN** all campaigns SHALL be sorted by the latest activity date regardless of their status

### Requirement: Frontend campaign list respects latest-first order
The student bidding view SHALL display campaigns in latest-first order, consistent with the API response ordering, even when campaigns are loaded across multiple paginated pages.

#### Scenario: Initial page load shows latest campaigns first
- **WHEN** a student opens the bidding view
- **THEN** campaigns SHALL be displayed with the most recently created or updated campaign at the top

#### Scenario: Infinite scroll preserves sort order
- **WHEN** a student scrolls down and loads additional pages of campaigns
- **THEN** newly loaded campaigns SHALL appear below previously loaded campaigns, maintaining the latest-first order within the full list

#### Scenario: Polling refresh updates order without page reload
- **WHEN** the polling interval triggers a data refresh and a campaign has been updated since the last fetch
- **THEN** the campaign list SHALL reflect the updated ordering automatically without requiring a manual page refresh

### Requirement: Active and completed campaigns visually separated
The student bidding view SHALL visually group campaigns into "Active" and "Completed" sections, with active campaigns displayed above completed campaigns.

#### Scenario: Active campaigns shown first
- **WHEN** a student has both active campaigns (with ongoing or upcoming phases) and completed campaigns (all phases ended)
- **THEN** active campaigns SHALL be displayed in a group above completed campaigns, each group maintaining latest-first ordering within itself

#### Scenario: Section headers displayed
- **WHEN** the student has campaigns in both groups
- **THEN** each group SHALL have a visible section header ("Active Campaigns" and "Completed Campaigns") to distinguish them

#### Scenario: Only active campaigns exist
- **WHEN** a student has no completed campaigns
- **THEN** only active campaigns SHALL be displayed without a "Completed Campaigns" section header

#### Scenario: Only completed campaigns exist
- **WHEN** a student has no active campaigns
- **THEN** only completed campaigns SHALL be displayed without an "Active Campaigns" section header

#### Scenario: Existing campaign data displayed correctly
- **WHEN** existing campaigns created before this change are loaded
- **THEN** they SHALL be sorted correctly using their existing `createdAt`/`updatedAt` timestamps with no data migration required
