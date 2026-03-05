## ADDED Requirements

### Requirement: Accurate pagination count for All Courses list

The `GET /flex-switch/list-course` endpoint must return accurate `pagination.total` and `pagination.total_pages` values that reflect the actual number of distinct class records matching the query filters, regardless of the number of JOIN-multiplied rows produced by the `class_promotions` relationship.

#### Scenario: Pagination total matches distinct class count
- **GIVEN** the database contains N distinct classes that match the applied filters
- **AND** some classes have multiple entries in the `class_promotions` table
- **WHEN** a PM requests `GET /flex-switch/list-course?page=1&limit=10`
- **THEN** `pagination.total` equals N (the count of distinct classes, not the count of class+promotion pairs)
- **AND** `pagination.total_pages` equals `ceil(N / 10)`

#### Scenario: Last page contains data
- **GIVEN** the database contains 6580 distinct classes matching the query
- **AND** the PM sets `limit=100`
- **WHEN** the PM navigates to page `ceil(6580 / 100)` = page 66
- **THEN** the response contains between 1 and 100 items
- **AND** `pagination.current_page` equals 66

#### Scenario: Pages beyond the last page are empty
- **GIVEN** the database contains 6580 distinct classes matching the query
- **AND** the PM sets `limit=100`
- **WHEN** the PM navigates to page 67 or higher
- **THEN** the response contains an empty `items` array
- **AND** `pagination.total` still equals 6580
- **AND** `pagination.total_pages` still equals 66

#### Scenario: Pagination with search filter
- **WHEN** a PM requests `GET /flex-switch/list-course?page=1&limit=100&search=Finance`
- **THEN** `pagination.total` equals the count of distinct classes whose name or section contains "Finance"
- **AND** navigating to the last page (as indicated by `total_pages`) returns data

#### Scenario: Pagination with credit/seat/campus filters
- **WHEN** a PM requests with credit_min, credit_max, seat_min, seat_max, or campus_ids filters
- **THEN** `pagination.total` equals the count of distinct classes matching ALL applied filters
- **AND** the last page as per `total_pages` returns data

#### Scenario: Export (no pagination) is unaffected
- **WHEN** a PM exports the course list (page/limit omitted)
- **THEN** all matching classes are returned without pagination
- **AND** no behavioral change occurs

### Requirement: PaginatedResult supports external count override

The `PaginatedResult` class must support an optional override for the total count, to allow callers to provide a more accurate count when Doctrine's `Paginator->count()` is inaccurate (e.g., due to GROUP BY with JOINs).

#### Scenario: Default behavior preserved
- **GIVEN** `PaginatedResult` is constructed without a total override
- **WHEN** `totalRecords` is accessed
- **THEN** it uses `Paginator->count()` as before (existing behavior unchanged)

#### Scenario: Override count used when provided
- **GIVEN** `PaginatedResult` is constructed with `totalOverride = 6580`
- **WHEN** `totalRecords` is accessed
- **THEN** `totalRecords` equals 6580
- **AND** `totalPages` equals `ceil(6580 / limit)`
