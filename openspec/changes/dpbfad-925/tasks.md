## 1. Backend — FlexSwitchService

- [x] 1.1 In `bidding-api/src/Service/FlexSwitch/FlexSwitchService.php`, locate the `submitRequest()` PM `createBulk()` call (~line 1014) and add `'action_url' => '/flex-switch/approval-request'` and `'action_label' => 'View Requests'` to its `$data` array
- [x] 1.2 In `FlexSwitchService::cancelRequest()`, locate the PM `createBulk()` call (~line 830) and add `'action_url' => '/flex-switch/approval-request'` and `'action_label' => 'View Requests'` to its `$data` array

## 2. Verification

- [ ] 2.1 Manually trigger a student switch submission and confirm the resulting PM notification has a non-null `action_url` in the database
- [ ] 2.2 Manually trigger a student switch cancellation and confirm the resulting PM notification has a non-null `action_url`
- [ ] 2.3 Open the notification centre as a PM user and confirm the "View Requests" button appears on switch notifications and navigates to `/flex-switch/approval-request`

