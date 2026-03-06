## ADDED Requirements

### Requirement: Calendar module deduplication by logical period

The PM Calendar View shall group calendar items by their logical identity (delivery mode + period number + campus) rather than by promotion period entity ID, eliminating duplicate module entries that contain the same courses.

#### Scenario: Multiple classPromotions with same period number produce single module

- **GIVEN** a class has two `classPromotion` records linked to different promotions (e.g., J25 and D25)
- **AND** both classPromotions have the same logical period number (e.g., period "1")
- **AND** both classPromotions share the same delivery mode and campus
- **WHEN** the PM requests the calendar view via `GET /flex-switch/calendar?program_id=X`
- **THEN** only one calendar item should appear for that delivery mode + period + campus combination (e.g., "Module 1: IN_PERSON FBL")
- **AND** the date range should span from the earliest `start_date` to the latest `end_date` across all matching classes

#### Scenario: Different period numbers remain separate

- **GIVEN** a class has classPromotions with period number "1" and another class has classPromotions with period number "2"
- **AND** both are IN_PERSON at the same campus
- **WHEN** the PM requests the calendar view
- **THEN** two separate calendar items appear: "Module 1: IN_PERSON FBL" and "Module 2: IN_PERSON FBL"

#### Scenario: Same period number but different delivery modes remain separate

- **GIVEN** classes exist with period number "1" at the same campus
- **AND** one is IN_PERSON and another is LIVE_VIRTUAL
- **WHEN** the PM requests the calendar view
- **THEN** two separate calendar items appear: "Module 1: IN_PERSON FBL" and "Module 1: LIVE_VIRTUAL FBL"

### Requirement: Repository program filter optimization

The `findCalendarClassesByProgram()` repository method shall filter classes by program at the database query level rather than loading all CORE classes and filtering in-memory.

#### Scenario: Only classes belonging to the requested program are returned

- **GIVEN** CORE classes exist for programs MBA (ID: 1) and EMBA (ID: 2)
- **WHEN** `findCalendarClassesByProgram(1)` is called
- **THEN** only classes linked to program ID 1 via `classPromotions → promotion → program` are returned
- **AND** classes belonging exclusively to program ID 2 are NOT returned

#### Scenario: API response shape is unchanged

- **GIVEN** the existing `FlexSwitchCalendarDto` response structure
- **WHEN** the fix is applied
- **THEN** the response JSON structure remains identical (array of `FlexSwitchCalendarGroupDto` with `program_name`, `program_id`, `campus_id`, `items[]`)
- **AND** only the duplicate items are removed; no new fields are added or existing fields removed
