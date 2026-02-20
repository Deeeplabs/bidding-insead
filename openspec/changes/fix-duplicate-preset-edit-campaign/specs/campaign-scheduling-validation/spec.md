# Campaign Scheduling Validation Specification

## Overview

This specification defines the validation rules for campaign phase dates to prevent overlapping dates between bidding rounds and simulation phases. The system SHALL validate that phase dates do not clash before allowing a campaign to run.

## ADDED Requirements

### Requirement: Bidding Round End Date Must Be Before Simulation Start Date

The system SHALL validate that the bidding round phase ends before the simulation phase starts.

#### Scenario: Bidding round ends before simulation starts (valid)
- **GIVEN** a campaign has a bidding round phase with end date "2026-03-01 17:00"
- **AND** the campaign has a simulation phase with start date "2026-03-01 18:00"
- **WHEN** the campaign is validated
- **THEN** no error SHALL be returned

#### Scenario: Bidding round overlaps with simulation (invalid)
- **GIVEN** a campaign has a bidding round phase with end date "2026-03-01 18:00"
- **AND** the campaign has a simulation phase with start date "2026-03-01 17:00"
- **WHEN** the campaign is validated
- **THEN** an error SHALL be returned: "Bidding round and simulation phases have overlapping dates"

#### Scenario: Bidding round ends after simulation starts (invalid)
- **GIVEN** a campaign has a bidding round phase with end date "2026-03-02 17:00"
- **AND** the campaign has a simulation phase with start date "2026-03-01 18:00"
- **WHEN** the campaign is validated
- **THEN** an error SHALL be returned: "Bidding round must end before simulation starts"

### Requirement: No Overlapping Dates Between Any Phases

The system SHALL validate that no two campaign phases have overlapping date ranges.

#### Scenario: Two phases with overlapping dates (invalid)
- **GIVEN** a campaign has phase A with dates "2026-02-01" to "2026-02-15"
- **AND** phase B with dates "2026-02-10" to "2026-02-20"
- **WHEN** the campaign is validated
- **THEN** an error SHALL be returned: "Phase 'Phase B' has overlapping dates with 'Phase A'"

#### Scenario: Sequential phases (valid)
- **GIVEN** a campaign has phase A with dates "2026-02-01" to "2026-02-15"
- **AND** phase B with dates "2026-02-16" to "2026-02-28"
- **WHEN** the campaign is validated
- **THEN** no error SHALL be returned

### Requirement: Validation Checked Before Campaign Can Start

The system SHALL run date validation when attempting to open/start a campaign.

#### Scenario: Campaign start blocked due to date overlap
- **GIVEN** a campaign has overlapping phase dates
- **WHEN** user attempts to change campaign status to "open"
- **THEN** the status change SHALL be blocked
- **AND** an error message SHALL be returned with details of the date conflict

## API Specification

### Campaign Start Validation

**Endpoint**: `POST /v2/api/campaigns/{id}/open`

**Response** (400 Bad Request):
```json
{
  "error": "validation_failed",
  "messages": [
    "Bidding round must end before simulation starts"
  ]
}
```

### Validation Service Method

```php
// CampaignValidationService
public function validatePhaseDateOverlap(Campaign $campaign): array
{
    // Returns array of error messages (empty if valid)
}
```

## Implementation Notes

- Add `validatePhaseDateOverlap()` method in `CampaignValidationService`
- Call this method in `validateCampaignStart()` to prevent campaign from opening with invalid dates
- Module codes to check: `bidding_round` and `simulation`
- Frontend should display validation errors in the campaign editing form
