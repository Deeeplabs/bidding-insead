## ADDED Requirements

### Requirement: PM switch notifications include action URL
When a Programme Manager receives a notification for a flex switch request event (submitted or cancelled), the notification SHALL include a populated `action_url` of `/flex-switch/approval-request` and an `action_label` of `View Requests`, enabling direct navigation to the approval request page.

#### Scenario: PM notification for switch submission includes action URL
- **WHEN** a student submits a flex switch request via `PUT /student/flex-switch/my-requests`
- **THEN** the PM notification created by `FlexSwitchService::submitRequest()` SHALL have `action_url = '/flex-switch/approval-request'` and `action_label = 'View Requests'` stored on the `Notification` entity

#### Scenario: PM notification for switch cancellation includes action URL
- **WHEN** a student cancels a flex switch request via `PUT /student/flex-switch/my-requests/{id}/cancel`
- **THEN** the PM notification created by `FlexSwitchService::cancelRequest()` SHALL have `action_url = '/flex-switch/approval-request'` and `action_label = 'View Requests'` stored on the `Notification` entity

#### Scenario: PM notification renders navigation button in UI
- **WHEN** a PM views a flex switch notification (submitted or cancelled) in the notification centre
- **THEN** the notification item SHALL display a "View Requests" button that navigates the PM to `/flex-switch/approval-request`

#### Scenario: Notifications without action URL are unaffected
- **WHEN** a `CUSTOM_ANNOUNCEMENT` notification is sent through any other code path that does not supply `action_url` in its data
- **THEN** those notifications SHALL NOT display an action button (existing behavior preserved)
