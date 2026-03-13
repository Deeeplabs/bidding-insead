# Fix: Student Dashboard Visibility for Different-Promotion Manual Additions

## Problem
Students who are manually added to a campaign for "Pre-bidding ONLY" from a different promotion cannot see the campaign on their dashboard or bidding page. The eligibility check only examines the currently active phase's student filters, missing students who were added in an earlier phase (e.g., pre-bidding) that is no longer active. Additionally, when impersonating these manually added students, all bidding phases (Pre-bidding, Add/Drop, Waitlist) are displayed instead of only the phases they were added to. There is also no restriction preventing admins from adding manual student inclusions to non-pre-bidding phases.

## Goal
- Make campaigns visible on the student dashboard for manually added cross-promotion students by checking student inclusion across all campaign phases, not just the active one.
- Filter the bidding phase list so cross-promotion students only see phases where they are explicitly included.
- Restrict manual student additions (`student_include` filter type) to the pre-bidding phase configuration only.

## Changes Made

1. **`src/Domain/Campaign/ActiveCampaign/CampaignStudentEligibilityService.php`**
   - Added `isStudentIncludedInAnyCampaignPhase(Student $student, Campaign $campaign): bool` — iterates all campaign executions and their phase configs to check if the student appears in any `student_include` filter across any phase (not just the active one). Checks both JSON config-based and table-based student filters.

2. **`src/Domain/Campaign/ActiveCampaign/ActiveCampaignService.php`**
   - Updated cross-promotion eligibility checks (both table-based and JSON config-based paths): after the standard `isStudentEligibleForCampaign()` check fails, falls back to `isStudentIncludedInAnyCampaignPhase()` before excluding the campaign.
   - Updated `getActiveBiddingRoundDetail()` security check: in the promotion-mismatch block, falls back to `isStudentIncludedInAnyCampaignPhase()` before throwing `AccessDeniedException`.
   - Updated module-level eligibility check: falls back to `isStudentIncludedInAnyCampaignPhase()` before denying access.

3. **`src/Domain/Campaign/Campaign/CampaignPhaseService.php`**
   - Added validation in `saveModuleConfigByCampaignAndModule()` that rejects `student_include` filter types on any module other than `pre_bidding`, throwing `InvalidArgumentException`.

4. **`src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapper.php`**
   - Updated `buildActiveBiddingRoundGroups()`: detects cross-promotion students (campaign promotion ≠ student promotion) and filters campaign modules to only those where the student is explicitly included via `student_include` filters.
   - Added private `isStudentIncludedInModule()` helper that checks both JSON config-based and table-based student filters for a specific module's phase config.

## Tests Added

5. **`tests/Unit/Domain/Campaign/ActiveCampaign/CampaignStudentEligibilityServiceTest.php`**
   - 4 tests: student found across multiple phases, student not found, deleted phases skipped, string ID matching.

6. **`tests/Unit/Domain/Campaign/Campaign/CampaignPhaseServiceManualAddRestrictionTest.php`**
   - 3 tests: `student_include` rejected on `add_drop_waitlist`, rejected on `bidding_round`, accepted on `pre_bidding`.

7. **`tests/Unit/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapperPhaseFilterTest.php`**
   - 2 tests: cross-promotion student sees only included phases, same-promotion student sees all phases.

## Impact
- **Student Dashboard API:** Cross-promotion students manually added for pre-bidding now correctly see the campaign listed and can access it.
- **Bidding Phase Display:** Cross-promotion students only see the specific phases they were included in (e.g., only Pre-bidding), instead of all phases.
- **Admin Configuration:** Attempting to add `student_include` filters to non-pre-bidding phase configs now returns a validation error.

## Testing / Verification Steps
- Impersonate a student from a different promotion who was manually added to a campaign's pre-bidding phase. Verify the campaign appears on their dashboard.
- Confirm the impersonated student only sees the Pre-bidding phase, not Add/Drop or Waitlist.
- Attempt to add a manual student inclusion to an Add/Drop or Bidding Round phase config via admin — verify it is rejected.
- Run PHPUnit: `php bin/phpunit --no-configuration tests/Unit/Domain/Campaign/ActiveCampaign/CampaignStudentEligibilityServiceTest.php tests/Unit/Domain/Campaign/Campaign/CampaignPhaseServiceManualAddRestrictionTest.php tests/Unit/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapperPhaseFilterTest.php` (9 tests, 13 assertions).
