## ADDED Requirements

### Requirement: Programme-Based Pre-Bidding Access Control
The system MUST validate that a student's Programme and Promotion match the campaign's Programme and Promotion before allowing access to pre-bidding pages.

#### Scenario: Authorized User Accessing Pre-Bidding
- **GIVEN** a student belongs to Programme "EMBA" and Promotion "EMBA-Jan2026"
- **AND** a campaign exists with Programme "EMBA" and Promotion "EMBA-Jan2026"
- **WHEN** the student navigates to the pre-bidding URL for that campaign
- **THEN** the system MUST allow access to the pre-bidding page
- **AND** the student MUST be able to view campaign details

#### Scenario: Unauthorized User Accessing Pre-Bidding via Direct URL
- **GIVEN** a student belongs to Programme "EMBA" and Promotion "EMBA-Jan2026"
- **AND** a campaign exists with Programme "MBA" and Promotion "MBA-Jan2026"
- **WHEN** the student directly navigates to the pre-bidding URL (e.g., by pasting the URL)
- **THEN** the system MUST deny access
- **AND** the student MUST be redirected to Access Denied page
- **AND** no campaign details, metadata, or bidding information MUST be exposed

#### Scenario: Unauthorized API Access
- **GIVEN** a student belongs to Programme "EMBA"
- **WHEN** the student makes an API request to `/api/student/campaign/{id}/pre-bidding` for an MBA campaign
- **THEN** the system MUST return 403 Forbidden response
- **AND** no campaign data MUST be included in the response

#### Scenario: Valid Access for Same Programme Different Promotion
- **GIVEN** a student belongs to Programme "MBA" and Promotion "MBA-Jan2026"
- **AND** a campaign exists with Programme "MBA" and Promotion "MBA-Jan2025"
- **WHEN** the student navigates to the pre-bidding URL for that campaign
- **THEN** the system MUST deny access (Promotion must match exactly)

#### Scenario: Missing Programme on Campaign
- **GIVEN** a campaign has no Programme assigned (null)
- **WHEN** any student navigates to the pre-bidding URL
- **THEN** the system SHOULD allow access (for backward compatibility with legacy data)

#### Scenario: URL Access Without Authentication
- **GIVEN** a user is not authenticated (not logged in)
- **WHEN** the user navigates to a pre-bidding URL
- **THEN** the system MUST redirect to login page
- **AND** no campaign data MUST be accessible
