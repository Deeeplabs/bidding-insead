## ADDED Requirements

### Requirement: SIS Integration Error Notification
The system MUST trigger an immediate notification when a synchronization mechanism with the SIS (PeopleSoft) fails for any student or course data.

#### Scenario: SIS sync failure caught by webhook or import service
- **WHEN** the system experiences an integration error or failure during SIS sync
- **THEN** it generates a Notification internally
- **AND** sets the notification message to: "Student or course data synchronization with the SIS has failed. Please investigate."
- **AND** addresses the notification to all active users with the Business Partner or Program Manager (PM) roles.
