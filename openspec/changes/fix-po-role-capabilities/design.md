## Context

The Programme Operations (PO) role was introduced as a secondary staff role alongside Programme Manager (PM). PO shares the same navigation (`staffNav`) and dashboard (`DashboardProgrammeManager`) as PM, but several capabilities are incomplete:

1. **Route access**: `role-modules.ts` defines which routes each role can access, enforced by `role-guard.ts` + `AuthGuards.tsx`. PO is missing `notification-centre` and `monitoring-analytics/audit-logs`.
2. **User Management page**: `user-management/page.tsx` only renders content for BP or PM — PO sees an empty page.
3. **Preset, Students**: Already functional for PO on the frontend.

**Existing permission architecture:**
- **Frontend**: `roleModules` in `role-modules.ts` -> `canAccessModule()` in `role-guard.ts` -> `AuthGuards.tsx` middleware
- **Backend**: `FeatureAccess` entity + `FeatureAccessVoter` + `RequireFeatureAccess` attribute -> database-driven per-role API access
- **User Management permissions**: `allowedImpersonate`, `allowedManageUser`, `allowedActivateUser` maps in `users-table.tsx` — PO not present as a key

## Goals / Non-Goals

**Goals:**
- PO can access Notification Centre (view-only)
- PO can access Audit Log page (view-only)
- PO can view User Management page showing PO users (view-only — no edit, delete, impersonate, or status toggle)
- Verify preset and student capabilities are fully functional for PO
- No changes to PM or BP capabilities

**Non-Goals:**
- Adding write/edit capabilities for PO in Notification or Audit Log
- Allowing PO to manage (edit/delete/impersonate) other users
- API-side data scoping (filtering audit logs or notifications to PO + Students only) — follow-up
- Changing FeatureAccess database seeds

## Decisions

### 1. Add modules to PO role in role-modules.ts

**File**: `bidding-admin/src/src/config/roles/role-modules.ts`

Add `'monitoring-analytics/audit-logs'` and `'notification-centre'` to `programme-operator`:

```typescript
'programme-operator': [
    'dashboard',
    'campaign',
    'preset',
    'campuses',
    'flex-switch',
    'course-list',
    'students',
    'user-management',
    'monitoring-analytics/audit-logs',  // NEW
    'notification-centre',              // NEW
],
```

This unblocks route access via `canAccessModule()`. The navigation already shows these items via `staffNav` — no nav changes needed.

### 2. Render User Management content for PO

**File**: `bidding-admin/src/app/(authenticated)/user-management/user-management/page.tsx`

Current code only checks `isBusinessPartner` and `isProgramManager`. Add a PO check and render `UserListGeneral` (which already defaults to filtering PM + PO users via `general-list.tsx`):

```typescript
const isProgrammeOperator = userData?.user_role === ROLE_ENUM.PROGRAMME_OPERATOR;
// ...
{(isProgramManager || isProgrammeOperator) && <UserListGeneral />}
```

### 3. Enforce view-only mode for PO in User Management

**File**: `bidding-admin/src/components/users/users-table.tsx`

PO should see user list but NOT have action capabilities. PO is not present in `allowedManageUser`, `allowedImpersonate`, or `allowedActivateUser` maps — so action buttons are already effectively disabled for PO because:
- `canManageUser()` returns false (PO not a key in `allowedManageUser`) — uses `?.includes`, safe
- Status toggle disabled (PO not a key in `allowedActivateUser`) — uses `?.includes`, safe
- Impersonate button: **required fix** — `allowedImpersonate[activeUser.role].includes(...)` was missing optional chaining. For PO, the map lookup returns `undefined` and `.includes()` throws a TypeError during render. Fixed by adding `?.includes` in 3 locations:
  - `handleImpersonate` guard condition (line 206)
  - Impersonate button `disabled` prop (line 373)
  - Impersonate button `className` prop (line 379)

### 4. User Management filter scope for PO

**File**: `bidding-admin/src/components/users/user-list/general-list.tsx`

`general-list.tsx` already filters by PM + PO roles by default:
```typescript
const rolePM = [
    roleList?.items.find((role) => role.name === ROLE_ENUM.PROGRAMME_MANAGER)?.id.toString()!,
    roleList?.items.find((role) => role.name === ROLE_ENUM.PROGRAMME_OPERATOR)?.id.toString()!,
];
```

For PO, we need to further restrict the default filter to show **only PO users** (not PM users). Add a role-based default:

```typescript
const { data: userData } = useUser();
const isPO = userData?.user_role === ROLE_ENUM.PROGRAMME_OPERATOR;

const rolePO = [
    roleList?.items.find((role) => role.name === ROLE_ENUM.PROGRAMME_OPERATOR)?.id.toString()!,
];

const defaultRoleIds = isPO ? rolePO : rolePM;
const finalRoleIds = params.role_ids.length > 0 ? params.role_ids : defaultRoleIds;
```

### 5. Filter dropdown for PO in User Management

**File**: `bidding-admin/src/components/users/filter/user-filter-dropdown.tsx`

Current filter already hides BP for PM users. For PO, hide both BP and PM:
```typescript
const isProgrammeOperator = userData?.user_role === ROLE_ENUM.PROGRAMME_OPERATOR;
// Filter roles: PO can only see PO role filter
```

### 6. No changes needed for Preset (Capability 3)

Preset page (`preset/page.tsx`) renders full CRUD without role checks. PO has `preset` in `roleModules`. All actions (Create, Edit, Delete, Duplicate, Toggle) are available. This is correct per the requirement.

### 7. No changes needed for Students (Capability 4)

Students page uses `menuActive='students'` which enables `['bulk-edit', 'impersonate']`. PO has `students` in `roleModules`. Impersonation works via `student-setting-table.tsx` `canImpersonate` prop. This is correct per the requirement.

## Risks / Trade-offs

- **Low risk**: Changes are frontend-only, additive module access for an existing role
- **API FeatureAccess**: If the API endpoints for notifications and audit logs have `RequireFeatureAccess` attribute, the PO role needs corresponding FeatureAccess entries in the database. This is a database seed concern, not a code change — verify in deployed environments
- **Data scoping**: Current implementation gives PO the same notification/audit-log data view as PM. If PO should only see PO + Student data, API-side filtering is needed as a follow-up
- **User Management view-only**: Relies on existing permission maps (`allowedManageUser` etc.) returning false for PO. If these maps change in the future, PO restrictions could be inadvertently loosened
