## ADDED Requirements

### Requirement: Duplicated Module Status Reset
The system MUST reset stateful tracking fields (such as actual start/end dates, active simulation phases, or filter statuses) when a module or bidding phase is duplicated, rather than copying the old states over.

#### Scenario: PM duplicates an active bidding round phase
- **WHEN** the PM duplicates a campaign or bidding phase that is currently active or finished (having an existing actual start/end date or status "filter")
- **THEN** the duplicated campaign/module is created with cleared manual state variables, reflecting only the expected configuration.

### Requirement: Display Actual vs Expected Dates Override
The frontend PM dashboard MUST dynamically show the module start/end dates. If a manual start/end override occurred, the output displays the real occurrence formatted with the predicted schedule.

#### Scenario: PM explicitly started a module manually earlier than scheduled
- **WHEN** the PM manually starts a module resulting in an actual start date differing from the expected start date
- **THEN** the UI lists the date as `Actual Start–End Date (Expected Start–End Date)`.

#### Scenario: Module has not been manually shifted
- **WHEN** the module's actual dates are either null or perfectly aligned with the expected dates
- **THEN** only the standard Expected Start-End Date is displayed without the parenthetical extra.
