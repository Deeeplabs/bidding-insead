## ADDED Requirements

### Requirement: Promotion count in Programme Governance Overview
The Programme Governance Overview section SHALL display the count of Promotion entities with `PromotionStatus::ACTIVE`. When a programme filter is applied, the count SHALL only include promotions belonging to the selected programme. The API response SHALL include a `promotions` field in the `programme_governance_overview` object.

#### Scenario: Promotion count without programme filter
- **WHEN** the BP Dashboard stats endpoint is called without a `programme_id` parameter
- **THEN** the `programme_governance_overview.promotions` field SHALL return the count of all Promotion entities where `status = ACTIVE`

#### Scenario: Promotion count with programme filter
- **WHEN** the BP Dashboard stats endpoint is called with a `programme_id` parameter
- **THEN** the `programme_governance_overview.promotions` field SHALL return the count of Promotion entities where `status = ACTIVE` AND `program.id = programme_id`

#### Scenario: Frontend displays promotion count
- **WHEN** the dashboard renders the Programme Governance Overview section
- **THEN** the "Promotion" metric card SHALL display the value from `programme_governance_overview.promotions`

#### Scenario: Backward compatibility of programme_operations field
- **WHEN** the BP Dashboard stats endpoint is called
- **THEN** the `programme_governance_overview.programme_operations` field SHALL continue to be returned with its existing Tier 2 ProgramManager count, unchanged

### Requirement: Total Users equals sum of role categories
The User Access Summary `total_users` field SHALL equal the sum of `business_partners + programme_managers + programme_operators + students`. This ensures the displayed total is mathematically consistent with the displayed per-role breakdown.

#### Scenario: Total users matches sum of categories
- **WHEN** the BP Dashboard stats endpoint returns the `user_access_summary` object
- **THEN** `total_users` SHALL equal `business_partners + programme_managers + programme_operators + students`

#### Scenario: All role categories are zero
- **WHEN** no users have any roles assigned
- **THEN** `total_users` SHALL be `0`

### Requirement: Students count matches active students
The User Access Summary `students` field SHALL count Student entities where `StudentStatus::ACTIVE` AND the student's promotion has `PromotionStatus::ACTIVE`. This aligns the dashboard count with the Students tab listing.

#### Scenario: Students count reflects active students with active promotions
- **WHEN** the BP Dashboard stats endpoint returns the `user_access_summary` object
- **THEN** `students` SHALL equal the count of Student entities where `status = ACTIVE` AND `promotion.status = ACTIVE`

#### Scenario: Inactive student not counted
- **WHEN** a student has `StudentStatus::INACTIVE` (e.g., deleted via DeleteStudentService)
- **THEN** that student SHALL NOT be included in the `students` count

#### Scenario: Student with inactive promotion not counted
- **WHEN** a student has `StudentStatus::ACTIVE` but their promotion has `PromotionStatus::INACTIVE`
- **THEN** that student SHALL NOT be included in the `students` count

### Requirement: API response shape for BP Dashboard stats
The `GET /dashboard/bp/stats` endpoint response SHALL include the following structure for `programme_governance_overview`:

```json
{
  "programme_governance_overview": {
    "active_campaigns": "<int>",
    "programme_managers": "<int>",
    "programme_operations": "<int>",
    "promotions": "<int>"
  }
}
```

The `promotions` field is a new addition. All existing fields remain unchanged.

#### Scenario: Response includes promotions field
- **WHEN** any consumer calls `GET /dashboard/bp/stats`
- **THEN** the response `programme_governance_overview` object SHALL include a `promotions` integer field alongside existing fields
