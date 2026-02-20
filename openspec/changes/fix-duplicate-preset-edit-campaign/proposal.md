## Why

There are **three** critical issues in the campaign management system:

1. **Duplicate Preset Issue**: When duplicating a campaign without selecting "Duplicate Campaign Flow" and "Duplicate All Configurations of Bidding Rounds", the preset from the original campaign is still being copied to the workflow page. This creates confusion for users who expect the preset to be empty when they are not duplicating the campaign flow.

2. **Missing Error Prevention**: When Bidding Round and Simulation date/time clash, the system currently allows users to proceed without validation. This leads to errors when running the campaign, forcing users to recreate/duplicate the campaign from scratch.

3. **False Preset Change Error**: When editing a campaign that is already "open", the system incorrectly blocks ALL saves even when the preset hasn't actually changed. Users receive an error message "Cannot change preset while campaign is open" even when they only edited other campaign settings.

## What Changes

### Issue 1 Fix - Duplicate Campaign Preset
- Modify campaign duplication logic to NOT copy preset when "Duplicate Campaign Flow" is not selected
- Ensure preset dropdown shows empty state when campaign flow is not duplicated
- Update both API (bidding-api) and Admin UI (bidding-admin) to handle preset clearing

### Issue 2 Fix - Date/Time Validation
- Add validation to prevent Bidding Round and Simulation date/time from overlapping
- Show clear error message to users when date/time conflicts are detected
- Prevent campaign from being saved/updated until conflicts are resolved
- Implement validation at both frontend (bidding-admin) and backend (bidding-api) levels

### Issue 3 Fix - Preset Edit Validation
- Modify campaign edit logic to compare existing preset_id with new preset_id
- Only block preset changes when: campaign status is "open" AND preset_id is actually different
- Allow saves when preset_id remains the same (no actual preset change)
- Fix false positive errors when editing other campaign settings on open campaigns

## Capabilities

### New Capabilities
- `campaign-duplication-preset`: Handle preset clearing during campaign duplication when campaign flow is not duplicated
- `campaign-scheduling-validation`: Validate that Bidding Round and Simulation dates do not overlap before allowing campaign to run
- `campaign-preset-edit-validation`: Fix preset edit validation to allow saves when preset_id hasn't actually changed on open campaigns

### Modified Capabilities
- `campaign-duplication`: Existing campaign duplication logic needs modification to handle preset clearing (existing spec: `dpbfad-684/specs/campaign-duplication/spec.md`)
- `campaign-phase-config`: May need to add validation rules for date/time conflicts

## Impact

### API (bidding-api)
- **Entity**: Campaign, CampaignPhaseConfig, Preset
- **Controllers**: Campaign duplication endpoints (likely in Api/Campaign/)
- **Domain Services**: Campaign duplication service logic
- **Validation**: New date/time overlap validation service

### Admin UI (bidding-admin)
- **Pages**: Campaign duplication workflow, Campaign editing
- **Components**: Preset selection component, Date/time picker with validation
- **Hooks**: Campaign duplication API hooks

### Database
- No new migrations required (validation logic only, no schema changes)

### Testing
- Integration tests for validation at API boundary
