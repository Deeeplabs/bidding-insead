# Fix: Add/Drop Config Sends Canonical Waitlist Limit Key

## Problem
Jira: [DPBFAD-718](https://insead.atlassian.net/browse/DPBFAD-718)

The Add/Drop & Waitlist configuration form saves the per-student waitlist limit as `waitlist_capacity` only. Several backend services expect the canonical key `max_waitlist_slots_per_student`, which the admin UI never sends. This config key mismatch is part of the root cause for the waitlist limit not being enforced.

## Changes Made

1. **`src/components/preset/configuration/add-drop-configuration.tsx`**
   - Added `max_waitlist_slots_per_student` to the save payload, set to the same value as `waitlist_capacity` when the "Enable Waitlist Capacity" toggle is ON, or `null` when OFF.
   - The existing `waitlist_capacity` field is preserved for backward compatibility.

## Depends On
- **bidding-api PR**: Fixes the backend to also fall back to `waitlist_capacity` for existing configs, wires the validator into the submission flow, and fixes the display logic. Both PRs should be deployed together.

## Testing / Verification Steps
1. Open a campaign preset or campaign module configuration for Add/Drop & Waitlist.
2. Enable "Waitlist Capacity" toggle and set limit to 5.
3. Save the configuration.
4. Verify in the API response / database that the module config contains both `waitlist_capacity: 5` and `max_waitlist_slots_per_student: 5`.
5. Disable the toggle, save again, and verify `max_waitlist_slots_per_student` is `null` in the saved config.
6. As a student, verify the Add/Drop page now shows the correct `Max waitlist allowed` value.
