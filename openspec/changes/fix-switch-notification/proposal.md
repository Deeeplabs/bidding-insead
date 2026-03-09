## Why

When a student cancels a flex switch request via `PUT /student/flex-switch/my-requests/{id}/cancel`, the PM (Program Manager) users do not receive any notification about the cancellation. This is a gap because PMs already receive notifications when a student **submits** a switch request (implemented in `FlexSwitchService::submitRequest`), but the corresponding `cancelRequest` method has no notification logic. PMs need to be informed when students cancel their requests so they can keep track of active requests accurately.

## What Changes

Add notification dispatch logic to `FlexSwitchService::cancelRequest()` to notify PM users when a student cancels their flex switch request. The implementation will follow the same pattern already used in `FlexSwitchService::submitRequest()` (lines 807-835), which:
1. Looks up the student's promotion → program → program managers
2. Sends a bulk notification to all PMs via `NotificationService::createBulk()`

## Capabilities

### New Capabilities
- `switch-cancel-pm-notification`: Trigger PM notification when a student cancels a flex switch request via the cancel API endpoint

### Modified Capabilities
_(none — no existing spec-level requirements are changing)_

## Impact

- **Service**: `App\Service\FlexSwitch\FlexSwitchService::cancelRequest()` — add notification trigger after successful cancellation (before returning result)
- **Dependencies**: Uses existing `NotificationService` (already injected), `UserRepository` (already injected), `ProgramManager` entity repository (already used in `submitRequest`)
- **API**: No response shape changes. The `PUT /student/flex-switch/my-requests/{id}/cancel` endpoint behavior is unchanged, only a side-effect (notification) is added
- **Migration**: None required — no schema changes
- **Breaking changes**: None
