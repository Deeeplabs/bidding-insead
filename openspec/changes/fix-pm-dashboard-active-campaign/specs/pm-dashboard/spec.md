## MODIFIED Requirements

### Requirement: Accurately count only currently open active campaigns
The PM dashboard needs to show the correct number of running active campaigns that have not passed their `endDate`. The API pagination metadata total count should exactly match the number of open filtered campaigns.

#### Scenario: PM views the dashboard header stats
- **WHEN** the PM lands on the dashboard
- **THEN** the total active campaigns count explicitly relies on a filtered query ensuring `status = open` and `endDate >= now`.

#### Scenario: PM pages through the active campaigns list
- **WHEN** the PM queries the campaign list API for open campaigns
- **THEN** both the data query limit results AND the metadata query return aligned counts resulting from enforcing the `endDate >= now` check.
