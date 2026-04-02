## Why

The Programme Manager configures a per-student waitlist limit (`waitlist_capacity`) in the Add/Drop & Waitlist module configuration, intended to cap how many courses a student may retain on the waitlist per bidding cycle. This limit is not enforced because of three interrelated bugs:

1. **Config key mismatch — admin saves vs backend reads**: The admin UI (`add-drop-configuration.tsx`) saves the per-student waitlist limit as `waitlist_capacity`, but most backend services look for `max_waitlist_slots_per_student` (which the admin never sends). Since `max_waitlist_slots_per_student` is always absent, enforcement code silently skips validation and display code falls through to 0.

2. **Validator never invoked**: `AddDropValidator::validateMaxWaitlistSlotsPerStudent()` exists but is never called from `AddDropService::submitAddDrop()`. Even if the config key were correct, the per-student waitlist limit would still not be enforced during Add/Drop submission.

3. **Display reads wrong module / wrong keys**: `EnrollmentViewService::getMaxWaitlistAllowed()` reads from the `final_enrollment` module instead of the `add_drop_waitlist` module. It also searches for keys (`max_waitlist_per_student`, `max_waitlist_allowed`) that the admin UI never sets. The method contains a dead-code bug (`$campaignConfig = null ?? []` — always empty). Result: the student sees `max_waitlist_allowed = 0` (or the hardcoded default 3) instead of the PM-configured value.

## What Changes

- Standardize reading of the per-student waitlist limit: all backend consumers will check `max_waitlist_slots_per_student` first, then fall back to `waitlist_capacity` for backward compatibility with existing saved configs.
- Wire `AddDropValidator::validateMaxWaitlistSlotsPerStudent()` into `AddDropService::submitAddDrop()` so the limit is actually enforced during student submissions.
- Fix `EnrollmentViewService::getMaxWaitlistAllowed()` to read from the `add_drop_waitlist` module and use the correct config keys.
- Fix `CampaignWaitlistService::getMaxWaitlistPerStudent()` to also fall back to `waitlist_capacity`.
- Update the admin UI to also send `max_waitlist_slots_per_student` alongside `waitlist_capacity` so new configs use the canonical key.

## Capabilities

### New Capabilities
- None — this is a bug fix restoring intended behavior.

### Modified Capabilities
- `waitlist-limit-enforcement`: Per-student waitlist limit configured by PM is now correctly enforced during Add/Drop submission and correctly displayed to students.

## Impact

- **bidding-api**: `AddDropService` (wire validator call), `AddDropValidator` (fix config key fallback), `CampaignToModuleDetailDtoMapper` (already has correct fallback — no change needed), `EnrollmentViewService` (fix module lookup + config keys), `CampaignWaitlistService` (fix config key fallback), `SimulationAlgorithmService` (already reads `waitlist_capacity` — no change needed)
- **bidding-admin**: `add-drop-configuration.tsx` (send `max_waitlist_slots_per_student` alongside `waitlist_capacity`)
- **bidding-web**: No changes needed — already reads `max_waitlist_allowed` from API response correctly
- **Entities affected**: None — no schema changes
- **No migration required**: This is a behavioral fix in PHP service logic and admin form payload
- **Backward compatibility**: Existing configs that store the limit as `waitlist_capacity` will continue to work via fallback. New configs will also set `max_waitlist_slots_per_student` for consistency.
