# Campaign Preset Edit Validation Specification

## Overview

This specification defines the validation rules for editing campaigns that are already "open". The system SHALL allow edits to open campaigns when the preset_id hasn't actually changed, while still blocking actual preset changes.

## ADDED Requirements

### Requirement: Allow Edits to Open Campaigns When Preset Unchanged

When editing a campaign that is already "open", the system SHALL allow the edit to proceed if the preset_id remains the same.

#### Scenario: Edit open campaign without changing preset (valid)
- **GIVEN** a campaign exists with status "open" and preset_id = 5
- **WHEN** user edits the campaign with `preset_id = 5` (same as current)
- **THEN** the edit SHALL succeed
- **AND** the campaign SHALL be updated with the new values

#### Scenario: Edit open campaign and change preset (invalid)
- **GIVEN** a campaign exists with status "open" and preset_id = 5
- **WHEN** user edits the campaign with `preset_id = 10` (different from current)
- **THEN** the edit SHALL be blocked
- **AND** an error message "Cannot change preset while campaign is open" SHALL be returned

#### Scenario: Edit open campaign without providing preset_id (valid - no change)
- **GIVEN** a campaign exists with status "open" and preset_id = 5
- **WHEN** user edits the campaign without providing preset_id in the request
- **THEN** the edit SHALL succeed (no preset change implied)
- **AND** the existing preset SHALL be preserved

#### Scenario: Edit draft campaign and change preset (valid)
- **GIVEN** a campaign exists with status "draft" and preset_id = 5
- **WHEN** user edits the campaign with `preset_id = 10`
- **THEN** the edit SHALL succeed
- **AND** the campaign SHALL have preset_id = 10

## API Specification

### Campaign Edit Endpoint

**Endpoint**: `PUT /v2/api/campaigns/{id}`

**Request Body**:
```json
{
  "campaign_name": "string|null",
  "preset_id": 5,
  "other_field": "new_value"
}
```

**Response** (200 OK):
```json
{
  "id": 123,
  "campaign_name": "Updated Campaign",
  "preset_id": 5,
  "status": "open"
}
```

**Response** (400 Bad Request - when changing preset on open campaign):
```json
{
  "error": {
    "code": "INVALID_ARGUMENT",
    "message": "Cannot change preset while campaign is open"
  }
}
```

## Implementation Notes

- The fix is in `CampaignService::updateCampaign()` or similar method
- Add comparison: `$existingPresetId = $campaign->getPreset()?->getId()`
- Only block if: `$campaign->getStatus() === 'open' && $existingPresetId != $newPresetId`
- This preserves the restriction on changing presets while allowing other edits
