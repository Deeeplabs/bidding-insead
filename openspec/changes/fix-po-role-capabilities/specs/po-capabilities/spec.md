## ADDED Requirements

### Requirement: PO can access Notification Centre

Programme Operations (PO) users must be able to access the Notification Centre page via the sidebar navigation. The page displays the same notification data as PM users.

#### Scenario: PO navigates to Notification Centre
- **GIVEN** a user logged in with Programme Operations role
- **WHEN** they click on "Notification" in the sidebar navigation
- **THEN** they are navigated to `/notification-centre`
- **AND** the page loads normally (not redirected to `/unauthorized`)
- **AND** the notification list is displayed with the same view as PM

#### Scenario: PO notification route is allowed by role guard
- **GIVEN** a PO user is authenticated
- **WHEN** `canAccessModule('programme-operator', '/notification-centre')` is called
- **THEN** it returns `true`

### Requirement: PO can access Audit Logs

PO users must be able to access the Monitoring & Analytics > Audit Logs page via the sidebar navigation.

#### Scenario: PO navigates to Audit Logs
- **GIVEN** a user logged in with Programme Operations role
- **WHEN** they click on "Audit Log" in the sidebar navigation
- **THEN** they are navigated to `/monitoring-analytics/audit-logs`
- **AND** the page loads normally (not redirected to `/unauthorized`)
- **AND** the audit log list is displayed with the same view as PM

#### Scenario: PO audit log route is allowed by role guard
- **GIVEN** a PO user is authenticated
- **WHEN** `canAccessModule('programme-operator', '/monitoring-analytics/audit-logs')` is called
- **THEN** it returns `true`

### Requirement: PO can view User Management with PO users only

PO users must see the User Management page with a list of PO users. The view is read-only — no edit, delete, impersonate, or status toggle actions.

#### Scenario: PO views User Management page
- **GIVEN** a user logged in with Programme Operations role
- **WHEN** they navigate to `/user-management/user-management`
- **THEN** the page renders the user list (not an empty page)
- **AND** the default filter shows only Programme Operations users
- **AND** the user list displays: Peoplesoft ID, Full Name, Email, Role, Status columns

#### Scenario: PO cannot edit users
- **GIVEN** a PO user is viewing the User Management page
- **WHEN** the user list is rendered
- **THEN** the Edit button is disabled or hidden for all listed users
- **AND** the Delete button is disabled or hidden for all listed users

#### Scenario: PO cannot impersonate users from User Management
- **GIVEN** a PO user is viewing the User Management page
- **WHEN** the user list is rendered
- **THEN** the Impersonate button is disabled for all listed users

#### Scenario: PO cannot toggle user status
- **GIVEN** a PO user is viewing the User Management page
- **WHEN** the user list is rendered
- **THEN** the status toggle switch is disabled for all listed users

#### Scenario: PO role filter in User Management
- **GIVEN** a PO user is viewing the User Management page
- **WHEN** they open the role filter dropdown
- **THEN** only the Programme Operations role option is available (BP and PM are hidden)

### Requirement: PO can edit and create presets

PO users must have full access to the Preset module, including creating, editing, duplicating, deleting, and toggling preset status.

#### Scenario: PO views preset list
- **GIVEN** a user logged in with Programme Operations role
- **WHEN** they navigate to `/preset`
- **THEN** the preset list page loads with all presets for the selected programme
- **AND** the "Create New Preset" button is visible and clickable

#### Scenario: PO creates a new preset
- **GIVEN** a PO user is on the preset list page
- **WHEN** they click "Create New Preset"
- **THEN** they are navigated to `/preset/create`
- **AND** the preset creation form is fully functional

#### Scenario: PO edits an existing preset
- **GIVEN** a PO user sees a preset in the list
- **WHEN** they click the Edit button on a preset
- **THEN** they are navigated to `/preset/edit?id={id}`
- **AND** the preset edit form is fully functional

### Requirement: PO can view and manage students

PO users must have access to the Students page with bulk-edit and impersonation capabilities.

#### Scenario: PO views student list
- **GIVEN** a user logged in with Programme Operations role
- **WHEN** they navigate to `/students`
- **THEN** the student list page loads with all students for the selected programme
- **AND** filter, search, and sort functionality works

#### Scenario: PO can impersonate a student
- **GIVEN** a PO user is on the Students page
- **WHEN** they click the Impersonate button on a student row
- **THEN** a confirmation modal appears
- **AND** upon confirming, the session switches to the student's account

#### Scenario: PO can bulk-edit students
- **GIVEN** a PO user selects multiple students on the Students page
- **WHEN** they click the "Bulk Edit" button
- **THEN** the bulk adjustment modal opens
- **AND** they can apply adjustments to the selected students

## UNCHANGED Requirements

### Requirement: PM capabilities remain unchanged

All existing PM capabilities must continue to work identically after PO capability changes.

#### Scenario: PM still has full access to all modules
- **GIVEN** a PM user is authenticated
- **WHEN** they navigate to any module (dashboard, campaign, preset, students, user-management, notification-centre, audit-logs)
- **THEN** access is granted as before with no behavioral changes

### Requirement: BP capabilities remain unchanged

All existing Business Partner capabilities must continue to work identically.

#### Scenario: BP still has admin-level access
- **GIVEN** a BP user is authenticated
- **WHEN** they navigate to any admin module
- **THEN** access is granted as before with no behavioral changes
