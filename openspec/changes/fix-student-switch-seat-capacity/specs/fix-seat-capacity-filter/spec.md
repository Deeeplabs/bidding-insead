## ADDED Requirements

### Requirement: Seat capacity filter uses aggregated total

The `GET /student/flex-switch/courses` endpoint's `seat_min` and `seat_max` filters must operate on the **total** (SUM) of `ClassPromotions.promotionSeats` across all ClassPromotions rows for a given class, not on individual row values.

#### Scenario: Filter by seat_min with multi-promotion class
- **GIVEN** a class with 3 ClassPromotions rows: 30, 30, 30 seats (total = 90)
- **WHEN** the student filters with `seat_min=50`
- **THEN** the class IS included in results (because SUM = 90 >= 50)
- **AND** the `seat_available` field shows `90`

#### Scenario: Filter by seat_max with multi-promotion class
- **GIVEN** a class with 3 ClassPromotions rows: 30, 30, 30 seats (total = 90)
- **WHEN** the student filters with `seat_max=50`
- **THEN** the class IS NOT included in results (because SUM = 90 > 50)

#### Scenario: Filter by seat_min excludes class below threshold
- **GIVEN** a class with 1 ClassPromotions row: 10 seats (total = 10)
- **WHEN** the student filters with `seat_min=20`
- **THEN** the class IS NOT included in results (because SUM = 10 < 20)

#### Scenario: Filter by seat_max includes class at boundary
- **GIVEN** a class with 2 ClassPromotions rows: 25, 25 seats (total = 50)
- **WHEN** the student filters with `seat_max=50`
- **THEN** the class IS included in results (because SUM = 50 <= 50)

#### Scenario: Combine seat_min and seat_max range
- **GIVEN** classes with total seats: Class A = 90, Class B = 30, Class C = 50
- **WHEN** the student filters with `seat_min=40` and `seat_max=60`
- **THEN** only Class C (50) is included in results

### Requirement: Pagination count is accurate with JOINs

The total count used for pagination must reflect the actual number of distinct classes matching filters, not inflated by JOINed rows.

#### Scenario: Count with multi-promotion class
- **GIVEN** 5 distinct classes, each having 3 ClassPromotions rows
- **WHEN** the student requests courses with no seat filter
- **THEN** `pagination.total_items` equals `5` (not 15)

#### Scenario: Count with seat filter applied
- **GIVEN** 5 distinct classes where 2 have total seats >= 50
- **WHEN** the student filters with `seat_min=50`
- **THEN** `pagination.total_items` equals `2`
- **AND** `pagination.total_pages` is calculated based on 2 total items
