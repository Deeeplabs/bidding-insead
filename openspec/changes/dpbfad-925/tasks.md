## 1. Root-cause fix — NotificationTemplateResolver

- [x] 1.1 In `bidding-api/src/Domain/Notification/NotificationTemplateResolver.php`, in the `render()` method (~line 73), change the `actionUrl` fallback guard from `$actionUrl === null` to `empty($actionUrl)` so that both `null` and `''` trigger the `$data['action_url']` fallback
- [x] 1.2 In the same method (~line 76), change the `actionLabel` fallback guard from `$actionLabel === null` to `empty($actionLabel)` so that both `null` and `''` trigger the `$data['action_label']` fallback

## 2. Data-passing — FlexSwitchService (already done)

- [x] 2.1 In `bidding-api/src/Service/FlexSwitch/FlexSwitchService.php`, `submitRequest()` PM `createBulk()` call (~line 1139) already includes `'action_url' => '/flex-switch/approval-request'` and `'action_label' => 'View Requests'` in `$data` — no change needed
- [x] 2.2 In `FlexSwitchService::cancelRequest()`, PM `createBulk()` call (~line 953) already includes `'action_url' => '/flex-switch/approval-request'` and `'action_label' => 'View Requests'` in `$data` — no change needed

## 3. Verification

- [ ] 3.1 Check the `notification_templates` table: `SELECT action_label, action_url_template FROM notification_templates WHERE notification_type_id = (SELECT id FROM notification_types WHERE code = 'CUSTOM_ANNOUNCEMENT')` — verify values are not empty string
- [ ] 3.2 Manually trigger a student switch submission and confirm the resulting PM notification has non-null `action_url` and `action_label` in the `notifications` table
- [ ] 3.3 Manually trigger a student switch cancellation and confirm the resulting PM notification has non-null `action_url` and `action_label`
- [ ] 3.4 Open the notification centre as a PM user and confirm the "View Requests" button appears on switch notifications and navigates to `/flex-switch/approval-request`

