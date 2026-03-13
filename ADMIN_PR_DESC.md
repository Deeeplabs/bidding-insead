# Fix Programme Operations (PO) Role Capabilities

## Problem

The Programme Operations / Faculty Support Coordinator (FSC) (Tier 2) role was missing several required capabilities:

1. **Notification Centre & Audit Logs blocked**: PO role was not included in `role-modules.ts` for `notification-centre` and `monitoring-analytics/audit-logs`, causing the role guard to redirect PO users to `/unauthorized` — even though the sidebar navigation displayed these menu items.

2. **User Management page empty**: `user-management/page.tsx` only rendered content for Business Partner (`isBusinessPartner`) and Programme Manager (`isProgramManager`). PO users saw a completely empty User Management page with no user list.

3. **User Management filter not scoped**: When PO gains access to User Management, the default filter should show only PO users (not PM + PO), and the role filter dropdown should only offer the PO role option.

## Solution

### Changes Made

**Modified Files:**

1. **`src/src/config/roles/role-modules.ts`**
   - Added `'monitoring-analytics/audit-logs'` and `'notification-centre'` to the `'programme-operator'` module list
   - PO now has the same module access as PM, unblocking Notification Centre and Audit Log routes

2. **`src/app/(authenticated)/user-management/user-management/page.tsx`**
   - Added `isProgrammeOperator` role detection using `ROLE_ENUM.PROGRAMME_OPERATOR`
   - Updated render condition: `{(isProgramManager || isProgrammeOperator) && <UserListGeneral />}`
   - PO now sees the same `UserListGeneral` component as PM (but with scoped filters)

3. **`src/components/users/user-list/general-list.tsx`**
   - Added `useUser` import and PO role detection
   - Added `rolePO` array containing only the PO role ID
   - Default role filter uses `rolePO` for PO users (only PO users shown) and `rolePM` for PM users (PM + PO shown)

4. **`src/components/users/filter/user-filter-dropdown.tsx`**
   - Added `isProgrammeOperator` detection
   - Updated role filter logic: PO sees only PO role option; PM sees all except BP; BP sees all roles

5. **`src/components/users/users-table.tsx`**
   - Added optional chaining (`?.includes`) to `allowedImpersonate` map lookups in 3 locations: `handleImpersonate` guard, impersonate button `disabled` prop, and impersonate button `className` prop
   - Without this fix, PO users viewing User Management would hit a runtime TypeError (`Cannot read properties of undefined`) because PO is not a key in the `allowedImpersonate` map
   - `canManageUser` and `allowedActivateUser` already used `?.includes` — this aligns the impersonate checks for consistency

## Capability Verification Summary

| # | Capability | Status |
|---|-----------|--------|
| 1 | Same view as PM (View only) | ✅ Fixed — added `notification-centre` + `monitoring-analytics/audit-logs` to PO modules |
| 2 | Notification + Audit log access | ✅ Fixed — PO can now access both pages via role guard |
| 3 | Preset edit/create | ✅ Already working — `preset` module enabled, no role restrictions on page |
| 4 | Students view + impersonation | ✅ Already working — `students` module enabled with `bulk-edit` + `impersonate` |
| 5 | User Management (view PO only) | ✅ Fixed — PO sees user list filtered to PO users, view-only (no edit/delete/impersonate actions) |

## Impact

- **No API changes**: Frontend-only fixes
- **No migration required**: No database schema changes
- **No breaking changes**: Purely additive — PM and BP capabilities remain unchanged
- **View-only enforcement**: PO is not present in `allowedManageUser`, `allowedImpersonate`, or `allowedActivateUser` maps in `users-table.tsx`, so action buttons are disabled for PO. Added optional chaining to `allowedImpersonate` lookups to prevent runtime TypeError for roles not present as map keys

## Verification

- [ ] Login as PO — confirm sidebar shows Notification and Audit Log menu items
- [ ] Navigate to `/notification-centre` as PO — page loads (not redirected to `/unauthorized`)
- [ ] Navigate to `/monitoring-analytics/audit-logs` as PO — page loads (not redirected to `/unauthorized`)
- [ ] Navigate to `/user-management/user-management` as PO — user list renders (not empty)
- [ ] Verify PO user list defaults to showing only PO users (not PM)
- [ ] Verify role filter dropdown for PO only shows Programme Operations option
- [ ] Verify Edit/Delete/Impersonate/Status toggle buttons are disabled for PO in User Management
- [ ] Navigate to `/preset` as PO — preset list loads with Create/Edit/Delete/Duplicate actions
- [ ] Navigate to `/students` as PO — student list loads with Impersonate and Bulk Edit actions
- [ ] Login as PM — verify all modules still work as before (regression check)
- [ ] Login as BP — verify all admin modules still work as before (regression check)
