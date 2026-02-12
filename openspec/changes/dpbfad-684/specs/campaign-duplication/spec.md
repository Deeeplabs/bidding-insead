## MODIFIED Requirements

### Requirement: Campaign duplication respects programme courses toggle
The system SHALL only duplicate course-related data (`courseSelection` JSON, `CampaignCourseFilter` records, and `AdjustmentCourse` records) when the `duplicateProgrammeCourses` flag is true. When the flag is false, the duplicated campaign SHALL have no course selection criteria, no course filters, and no adjustment courses.

#### Scenario: Duplicate campaign with programme courses unchecked
- **WHEN** a PM duplicates a campaign with `duplicateProgrammeCourses` set to false
- **THEN** the duplicated campaign SHALL NOT contain the source campaign's `courseSelection` JSON
- **THEN** the duplicated campaign SHALL NOT contain any `CampaignCourseFilter` records from the source campaign
- **THEN** the duplicated campaign SHALL NOT contain any `AdjustmentCourse` records from the source campaign
- **THEN** the duplicated campaign's course list SHALL be empty

#### Scenario: Duplicate campaign with programme courses checked
- **WHEN** a PM duplicates a campaign with `duplicateProgrammeCourses` set to true
- **THEN** the duplicated campaign SHALL contain the same `courseSelection` JSON as the source campaign
- **THEN** the duplicated campaign SHALL contain copies of all `CampaignCourseFilter` records from the source campaign
- **THEN** the duplicated campaign SHALL contain copies of all `AdjustmentCourse` records (including conflicts and fallbacks) from the source campaign

#### Scenario: Main config duplication is independent of course data
- **WHEN** a PM duplicates a campaign with `duplicateMainConfig` set to true and `duplicateProgrammeCourses` set to false
- **THEN** the duplicated campaign SHALL copy main config fields (dates, credits, capital, description, student filters)
- **THEN** the duplicated campaign SHALL NOT copy any course-related data (`courseSelection`, `CampaignCourseFilter`, `AdjustmentCourse`)

#### Scenario: Other duplication flags remain unaffected
- **WHEN** a PM duplicates a campaign with any combination of `duplicateCampaignFlow` and `duplicateBiddingRounds` flags
- **THEN** campaign flow and bidding round duplication behavior SHALL remain unchanged from current behavior
