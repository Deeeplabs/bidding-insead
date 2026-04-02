## Context

The PM configures a per-student waitlist limit in the Add/Drop & Waitlist module via the admin dashboard. The admin form (`add-drop-configuration.tsx`) saves this as `waitlist_capacity` in the module config. The API request DTO (`AddDropWaitlistConfigRequest`) supports both `waitlist_capacity` and `max_waitlist_slots_per_student`, but the admin only sends `waitlist_capacity`.

The config is stored in `CampaignPhaseConfig.moduleConfig` as:
```json
{
  "config": {
    "limit_waitlist_capacity": true,
    "waitlist_capacity": 5
  }
}
```

Multiple backend services read this value using different keys, causing inconsistent behavior:

| Service | Key(s) Read | Module Source | Result |
|---------|-------------|--------------|--------|
| `CampaignToModuleDetailDtoMapper` (L803, L1231) | `max_waitlist_slots_per_student` ?? `waitlist_capacity` | Add/Drop phase | Works (fallback hits) |
| `AddDropValidator::validateMaxWaitlistSlotsPerStudent` (L435) | `max_waitlist_slots_per_student` only | Add/Drop config | Always null → skips validation |
| `EnrollmentViewService::getMaxWaitlistAllowed` (L586) | `max_waitlist_per_student`, `max_waitlist_allowed` | `final_enrollment` (wrong module!) | Never finds value → returns default 3 |
| `CampaignWaitlistService::getMaxWaitlistPerStudent` (L580) | `max_waitlist_slots_per_student` ?? `max_waitlist_per_student` | Active phase | Falls through → returns 0 |
| `SimulationAlgorithmService::allocateCourseStudents` (L994) | `waitlist_capacity` | Add/Drop config | Works correctly |

Additionally, `AddDropValidator::validateMaxWaitlistSlotsPerStudent()` is defined but **never called** from `AddDropService::submitAddDrop()`, so even if the config key were correct, the limit would not be enforced.

## Goals / Non-Goals

**Goals:**
- Enforce the PM-configured per-student waitlist limit during Add/Drop submission
- Display the correct `max_waitlist_allowed` value to students across all views (Add/Drop page, Final Enrollment page)
- Standardize config key resolution: `max_waitlist_slots_per_student` → `waitlist_capacity` fallback
- Ensure the admin UI sends the canonical key `max_waitlist_slots_per_student` for new configs

**Non-Goals:**
- Changing the simulation algorithm's waitlist handling (already works correctly)
- Renaming existing config keys in the database (backward compatibility)
- Modifying the class-level waitlist capacity logic (separate concern)
- Changing the student portal UI (already reads `max_waitlist_allowed` correctly)

## Decisions

### Decision 1: Fix config key resolution with fallback chain

**Approach**: All backend services that read the per-student waitlist limit will use the same resolution order:
1. `$config['max_waitlist_slots_per_student']` (canonical key)
2. `$config['waitlist_capacity']` (legacy key from admin UI)
3. `0` or `null` (default — no limit)

**Files affected:**
- `AddDropValidator::validateMaxWaitlistSlotsPerStudent()` — change `$moduleConfig['config']['max_waitlist_slots_per_student'] ?? null` to `$moduleConfig['config']['max_waitlist_slots_per_student'] ?? $moduleConfig['config']['waitlist_capacity'] ?? null`
- `CampaignWaitlistService::getMaxWaitlistPerStudent()` — add `waitlist_capacity` to the fallback chain
- `CampaignToModuleDetailDtoMapper` — already correct (has `waitlist_capacity` fallback), no change needed

**Why fallback instead of rename?**
- Existing campaign configs in the database store the value as `waitlist_capacity`
- Changing stored data would require a data migration affecting live campaigns
- Fallback is safe and handles both old and new configs

### Decision 2: Wire validator into AddDropService

**Approach**: In `AddDropService::submitAddDrop()`, call `$this->validator->validateMaxWaitlistSlotsPerStudent()` after the existing enrollment validations (Step 5) and before executing the add/drop (Step 11). Pass the `$moduleConfig` array which contains the Add/Drop phase config.

The call should respect the `limit_waitlist_capacity` toggle: only validate when the toggle is enabled.

**Where in the flow**: After Step 8a (validateBidPoints) and before Step 11 (executeCampaignAddDrop), approximately around line 194 in `AddDropService.php`. This ensures all other validations pass first.

**Why not earlier?** The validator needs to know which enrollments will become waitlisted (classes that are full). This information is available at this point in the flow.

### Decision 3: Fix EnrollmentViewService to read from correct module

**Approach**: Replace the `final_enrollment` module lookup with `add_drop_waitlist`:
```
$activePhase = $campaign->getCurrentActivePhaseDetailsByModule('add_drop_waitlist');
```

And fix the config key resolution to use the standard fallback chain:
```
$config['max_waitlist_slots_per_student'] ?? $config['waitlist_capacity'] ?? 0
```

Also remove the dead-code line `$campaignConfig = null ?? []` and the module iteration fallback (which uses wrong keys). Replace with a clean, single-source lookup.

### Decision 4: Admin UI sends canonical key alongside legacy key

**Approach**: In `add-drop-configuration.tsx`, when building the save payload, also include `max_waitlist_slots_per_student` set to the same value as `waitlist_capacity`. This ensures new configs stored going forward contain the canonical key.

```tsx
max_waitlist_slots_per_student: values.limit_waitlist_capacity ? values.waitlist_capacity : null,
```

**Why keep both keys?** `waitlist_capacity` is still needed for backward compatibility with existing code paths (e.g., `SimulationAlgorithmService`). Sending both ensures all consumers work regardless of which key they read first.

### Decision 5: Respect the limit_waitlist_capacity toggle

The `limit_waitlist_capacity` boolean controls whether the waitlist limit is active. The validator and display logic must respect this:
- If `limit_waitlist_capacity === false` (or not set): no enforcement, display `max_waitlist_allowed = 0` (unlimited)
- If `limit_waitlist_capacity === true`: enforce the configured limit

The `AddDropValidator::validateMaxWaitlistSlotsPerStudent()` should check the toggle before enforcing. Currently it only checks if the value is null, but it should also check the toggle.

## Risks / Trade-offs

- **[Risk] Existing campaigns with misconfigured limits** → Campaigns where `waitlist_capacity` was set but `limit_waitlist_capacity` was false will continue to have no enforcement (correct behavior). Campaigns where the toggle is on but the admin saved only `waitlist_capacity` will now correctly enforce the limit (desired fix).
- **[Risk] SimulationAlgorithmService reads `waitlist_capacity` directly** → This service already works correctly with the current config. No change needed, but if someone later refactors to use only `max_waitlist_slots_per_student`, the simulation would break. The admin sending both keys mitigates this.
- **[Trade-off] Dual config keys** → More complex config resolution, but necessary for zero-downtime backward compatibility. Over time, old configs can be migrated to use the canonical key.
- **[Low risk] EnrollmentViewService module change** → Changing from `final_enrollment` to `add_drop_waitlist` could affect the Final Enrollment page display. However, the waitlist limit is semantically an Add/Drop concern, so reading from that module is correct.
