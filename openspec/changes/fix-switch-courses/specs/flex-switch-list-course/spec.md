## ADDED Requirements

### Requirement: Array query parameters handled correctly in FlexSwitch endpoints

The `FlexSwitchController` must correctly handle array-type query parameters (`campus_ids`, `modes`) sent using PHP array notation (e.g., `campus_ids[]=2&campus_ids[]=3`) without throwing a 400 Bad Request error. This applies to `listCourse()`, `listClassConfiguration()`, and `moduleDetail()`.

#### Scenario: Single campus filter via array notation
- **GIVEN** a PM is viewing the All Courses list
- **WHEN** the PM requests `GET /flex-switch/list-course?page=1&limit=10&programmeId=2&campus_ids[]=2`
- **THEN** the response returns `success: true`
- **AND** all returned items have `campus` matching campus ID 2
- **AND** pagination reflects the filtered count

#### Scenario: Multiple campus filters via array notation
- **GIVEN** a PM is viewing the All Courses list
- **WHEN** the PM requests `GET /flex-switch/list-course?page=1&limit=10&campus_ids[]=2&campus_ids[]=3`
- **THEN** the response returns `success: true`
- **AND** all returned items have `campus` matching either campus ID 2 or 3

#### Scenario: No campus filter provided
- **GIVEN** a PM is viewing the All Courses list
- **WHEN** the PM requests `GET /flex-switch/list-course?page=1&limit=10` (no `campus_ids` parameter)
- **THEN** the response returns `success: true`
- **AND** items from all campuses are returned (no campus filtering applied)

#### Scenario: Mode filter via array notation
- **GIVEN** a PM is viewing the All Courses list
- **WHEN** the PM requests `GET /flex-switch/list-course?page=1&limit=10&modes[]=online`
- **THEN** the response returns `success: true`
- **AND** all returned items have the matching delivery mode

#### Scenario: Combined campus and mode filters
- **WHEN** the PM requests `GET /flex-switch/list-course?page=1&limit=10&campus_ids[]=2&modes[]=online`
- **THEN** the response returns `success: true`
- **AND** items are filtered by both campus ID 2 AND delivery mode "online"

#### Scenario: Backward compatibility with comma-separated string format
- **WHEN** a client requests `GET /flex-switch/list-course?page=1&limit=10&campus_ids=2,3`
- **THEN** the response returns `success: true`
- **AND** items are filtered by campus IDs 2 and 3

#### Scenario: Class configuration endpoint with array campus filter
- **WHEN** the PM requests `GET /flex-switch-class-configuration?program_id=1&campus_ids[]=2`
- **THEN** the response returns successfully with campus-filtered results

#### Scenario: Module detail endpoint with array campus filter
- **WHEN** the PM requests `GET /flex-switch/{id}/{mode}/{campus}/module-detail?campus_ids[]=2`
- **THEN** the response returns successfully with campus-filtered results
