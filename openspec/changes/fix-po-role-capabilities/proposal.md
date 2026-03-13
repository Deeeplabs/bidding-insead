## Why

Programme Operations (PO) / Faculty Support Coordinator (FSC) (Tier 2) role was introduced alongside Programme Manager (PM) (Tier 1) to provide a secondary staff role with specific view/action permissions. However, a verification audit of the 5 required PO capabilities reveals that **3 out of 5 are not fully resolved** in the current codebase.

### Current State Summary

| # | Capability | Status | Root Cause |
|---|-----------|--------|------------|
| 1 | Same view as PM (View only) | ⚠️ Partial | Navigation uses same `staffNav`, but `role-modules.ts` blocks PO from `notification-centre` and `monitoring-analytics/audit-logs` |
| 2 | Notification + Audit log (view PO + Students) | ❌ Not Resolved | PO missing from `role-modules.ts` for both modules; role guard redirects to `/unauthorized` |
| 3 | Can PO edit/create presets | ✅ Resolved (frontend) | `preset` module is in PO's role-modules; preset page has no role-gated restrictions. API FeatureAccess needs DB verification |
| 4 | Students — view list + actions (impersonation) | ✅ Resolved (frontend) | `students` module enabled; `menuActive='students'` enables `['bulk-edit', 'impersonate']` |
| 5 | User Management — only view PO | ❌ Not Resolved | `user-management/page.tsx` only renders content for BP or PM; PO sees **empty page** |

### Detailed Findings

**Capability 1 — Same view as PM (View only):**
- `navigation.tsx` line 256-258: PO uses same `staffNav` as PM — same sidebar items rendered
- `dashboard/page.tsx` line 13: PO uses same `DashboardProgrammeManager` component
- `role-modules.ts` line 22-31: PO has `dashboard, campaign, preset, campuses, flex-switch, course-list, students, user-management` but **missing** `monitoring-analytics/audit-logs` and `notification-centre`
- `role-guard.ts` + `AuthGuards.tsx`: `canAccessModule()` blocks PO from accessing missing module routes → redirects to `/unauthorized`
- **Inconsistency**: Nav shows Notification and Audit Log items to PO, but guard blocks access

**Capability 2 — Notification + Audit log:**
- PO should have view-only access to notifications and audit logs scoped to PO + Students
- Currently completely blocked — not in `roleModules['programme-operator']`

**Capability 3 — Preset edit/create:**
- `role-modules.ts` includes `preset` for PO ✅
- `preset/page.tsx` renders full CRUD (Create, Edit, Delete, Duplicate, Toggle) without role checks ✅
- API-side: FeatureAccess voter checks are database-driven — needs DB seed verification

**Capability 4 — Students view + actions:**
- `role-modules.ts` includes `students` for PO ✅
- `students/page.tsx` uses `StudentList` with `menuActive='students'`
- `showBasedMenu['students']` = `['bulk-edit', 'impersonate']` — both enabled ✅
- Student impersonation works via `student-setting-table.tsx` `canImpersonate` prop ✅

**Capability 5 — User Management (view PO only):**
- `user-management/page.tsx` line 13-14: `isBusinessPartner` and `isProgramManager` checks — **PO matches neither**
- Line 20-21: Content only renders for BP (`UserListManagementAdmin`) or PM (`UserListGeneral`)
- PO sees an **empty User Management page** — no user list rendered
- `general-list.tsx` already filters by PM + PO roles by default — just never rendered for PO
- `allowedManageUser`, `allowedImpersonate`, `allowedActivateUser` in `users-table.tsx` — PO not listed as a key (cannot manage/impersonate other users)

## What Changes

### Fix 1: Add notification-centre and audit-logs to PO role modules
- Add `'monitoring-analytics/audit-logs'` and `'notification-centre'` to `roleModules['programme-operator']` in `role-modules.ts`
- This unblocks route access via the role guard

### Fix 2: Render User Management content for PO
- Update `user-management/page.tsx` to detect PO role and render `UserListGeneral` (same as PM) in view-only mode
- PO should see PO users only — `general-list.tsx` already defaults to PM + PO role filter
- PO should NOT have edit/delete/impersonate/activate actions on users — view-only

### Fix 3: (Optional) Scope notification and audit log data for PO
- If PO should only see notifications/audit logs related to PO + Students (per requirement #2), API-side filtering may be needed
- This depends on whether the existing API endpoints already filter by user role or return all data

## Capabilities

### New Capabilities
- `po-notification-access`: PO can view the Notification Centre (view-only, scoped to PO + Students)
- `po-audit-log-access`: PO can view the Audit Log page (view-only, scoped to PO + Students)
- `po-user-management-view`: PO can view User Management page showing only PO users (view-only, no edit/delete/impersonate)

### Modified Capabilities
- `po-role-modules`: Updated module access list for PO role

## Impact

- **Admin (bidding-admin)**:
  - `src/src/config/roles/role-modules.ts` — Add 2 modules to PO role
  - `src/app/(authenticated)/user-management/user-management/page.tsx` — Add PO condition to render user list
  - Optionally: `src/components/users/users-table.tsx` — Enforce view-only for PO (hide action buttons)
- **API (bidding-api)**: Possibly scope notification/audit-log API responses by role (if not already filtered)
- **No migration required** — No schema changes
- **No breaking changes** — Purely additive behavior for PO role
