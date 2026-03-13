## 1. Add Notification Centre and Audit Logs to PO Role Modules

- [x] 1.1 **Update `role-modules.ts` to add missing modules for PO**
  - File: `bidding-admin/src/src/config/roles/role-modules.ts`
  - Add `'monitoring-analytics/audit-logs'` and `'notification-centre'` to the `'programme-operator'` array
  - After change, PO should have the same modules as PM

## 2. Fix User Management Page for PO

- [x] 2.1 **Add PO role detection in User Management page**
  - File: `bidding-admin/src/app/(authenticated)/user-management/user-management/page.tsx`
  - Add `const isProgrammeOperator = userData?.user_role === ROLE_ENUM.PROGRAMME_OPERATOR;`
  - Update render condition: `{(isProgramManager || isProgrammeOperator) && <UserListGeneral />}`

- [x] 2.2 **Restrict default role filter for PO in general-list.tsx**
  - File: `bidding-admin/src/components/users/user-list/general-list.tsx`
  - Import `useUser` and detect PO role
  - When PO: default `finalRoleIds` to only PO role IDs (not PM + PO)
  - When PM: keep existing behavior (PM + PO role IDs)

- [x] 2.3 **Restrict role filter dropdown for PO**
  - File: `bidding-admin/src/components/users/filter/user-filter-dropdown.tsx`
  - Add PO detection: `const isProgrammeOperator = userData?.user_role === ROLE_ENUM.PROGRAMME_OPERATOR;`
  - When PO: filter role options to show only PO role (hide BP and PM)
  - When PM: keep existing behavior (hide BP only)

- [x] 2.4 **Fix optional chaining for impersonate permission check in users-table.tsx**
  - File: `bidding-admin/src/components/users/users-table.tsx`
  - `allowedImpersonate[activeUser.role].includes(...)` throws TypeError for PO (not a key in the map)
  - Add `?.includes` in 3 locations: `handleImpersonate` guard (line 206), button `disabled` prop (line 373), button `className` prop (line 379)
  - `canManageUser` and `allowedActivateUser` already used `?.includes` — this aligns the impersonate checks

## 3. Verification — Capability 1: Same View as PM

- [ ] 3.1 **Verify PO sees same navigation as PM**
  - Login as PO user
  - Confirm sidebar shows: Dashboard, Notification, Campaign, Preset, Switch, Settings (Students, Courses, User Management), Audit Log
  - All items should be clickable and navigate to their respective pages

- [ ] 3.2 **Verify PO dashboard matches PM dashboard**
  - Login as PO user
  - Navigate to `/dashboard`
  - Confirm `DashboardProgrammeManager` component renders (same as PM view)

## 4. Verification — Capability 2: Notification + Audit Log Access

- [ ] 4.1 **Verify PO can access Notification Centre**
  - Login as PO user
  - Navigate to `/notification-centre`
  - Confirm page loads (not redirected to `/unauthorized`)
  - Confirm notification list is displayed

- [ ] 4.2 **Verify PO can access Audit Logs**
  - Login as PO user
  - Navigate to `/monitoring-analytics/audit-logs`
  - Confirm page loads (not redirected to `/unauthorized`)
  - Confirm audit log list is displayed

- [ ] 4.3 **Verify API FeatureAccess for PO notification/audit-log endpoints**
  - Check if notification list API (`GET /v2/api/notifications`) has `RequireFeatureAccess` attribute
  - Check if audit log list API (`GET /v2/api/audit-logs`) has `RequireFeatureAccess` attribute
  - If yes, verify PO role has corresponding FeatureAccess entries in the database
  - If API returns 403, add FeatureAccess DB seeds for PO role (separate migration task)

## 5. Verification — Capability 3: Preset Edit/Create

- [ ] 5.1 **Verify PO can view preset list**
  - Login as PO user
  - Navigate to `/preset`
  - Confirm preset list loads with "Create New Preset" button visible

- [ ] 5.2 **Verify PO can create a new preset**
  - Click "Create New Preset"
  - Fill out the form and submit
  - Confirm preset is created successfully

- [ ] 5.3 **Verify PO can edit/duplicate/delete presets**
  - Click Edit on an existing preset — confirm edit form loads
  - Click Duplicate — confirm duplicate modal and action works
  - Click Delete on an unused preset — confirm delete works
  - Toggle preset status — confirm status changes

## 6. Verification — Capability 4: Students View + Actions

- [ ] 6.1 **Verify PO can view student list**
  - Login as PO user
  - Navigate to `/students`
  - Confirm student list loads with filter/search/sort functionality

- [ ] 6.2 **Verify PO can impersonate a student**
  - Click the Impersonate button on a student row
  - Confirm confirmation modal appears
  - Confirm session switches to student account upon confirming

- [ ] 6.3 **Verify PO can bulk-edit students**
  - Select multiple students
  - Click "Bulk Edit" button
  - Confirm bulk adjustment modal opens and adjustments can be applied

## 7. Verification — Capability 5: User Management View-Only

- [ ] 7.1 **Verify PO sees user list (not empty page)**
  - Login as PO user
  - Navigate to `/user-management/user-management`
  - Confirm user list renders with PO users

- [ ] 7.2 **Verify PO default filter shows only PO users**
  - Confirm the user list defaults to showing only Programme Operations users
  - Confirm PM and BP users are not shown by default

- [ ] 7.3 **Verify PO actions are disabled (view-only)**
  - Confirm Edit button is disabled/hidden for all listed users
  - Confirm Delete button is disabled/hidden for all listed users
  - Confirm Impersonate button is disabled for all listed users
  - Confirm Status toggle is disabled for all listed users

- [ ] 7.4 **Verify PO role filter only shows PO option**
  - Open the role filter dropdown
  - Confirm only Programme Operations role is available
  - Confirm BP and PM roles are not visible in the dropdown

## 8. Regression — No Impact on PM and BP

- [ ] 8.1 **Verify PM capabilities unchanged**
  - Login as PM user
  - Navigate through all modules: dashboard, campaign, preset, students, user-management, notification-centre, audit-logs
  - Confirm all work as before

- [ ] 8.2 **Verify BP capabilities unchanged**
  - Login as BP user
  - Navigate through all admin modules
  - Confirm all work as before
