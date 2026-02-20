# Campaign Duplication Preset Specification

## Overview

This specification defines the behavior for handling preset when duplicating a campaign. The preset should only be copied when the campaign flow (workflow modules) is being duplicated.

## ADDED Requirements

### Requirement: Preset Not Copied When Campaign Flow Is Not Duplicated

When duplicating a campaign with `duplicate_campaign_flow=false`, the system SHALL NOT copy the preset from the source campaign. The duplicated campaign SHALL have a null/empty preset.

#### Scenario: Duplicate campaign without campaign flow
- **GIVEN** a campaign exists with a preset assigned
- **WHEN** user duplicates the campaign with `duplicate_campaign_flow=false`
- **THEN** the duplicated campaign SHALL NOT have a preset assigned
- **AND** the preset dropdown in the workflow page SHALL be empty

#### Scenario: Duplicate campaign with campaign flow
- **GIVEN** a campaign exists with a preset assigned
- **WHEN** user duplicates the campaign with `duplicate_campaign_flow=true`
- **THEN** the duplicated campaign SHALL have the same preset as the source campaign

#### Scenario: Duplicate campaign flow false with bidding rounds true
- **GIVEN** a campaign exists with a preset and bidding round configurations
- **WHEN** user duplicates the campaign with `duplicate_campaign_flow=false` and `duplicate_bidding_rounds=true`
- **THEN** the duplicated campaign SHALL NOT have a preset assigned
- **AND** the bidding round configurations SHALL be duplicated

## API Specification

### Campaign Duplication Endpoint

**Endpoint**: `POST /v2/api/campaigns/{id}/duplicate`

**Request Body**:
```json
{
  "new_campaign_name": "string|null",
  "duplicate_main_config": true,
  "duplicate_campaign_flow": false,
  "duplicate_programme_courses": true,
  "duplicate_bidding_rounds": true
}
```

**Response** (201 Created):
```json
{
  "id": 123,
  "campaign_name": "Duplicated Campaign",
  "preset": null,
  "status": "draft"
}
```

## Implementation Notes

- The fix is in `CampaignDuplicationService::duplicateCampaign()` method
- Move `$newCampaign->setPreset($sourceCampaign->getPreset())` inside the `if ($duplicateCampaignFlow)` block
- This preserves backward compatibility when `duplicate_campaign_flow=true`
