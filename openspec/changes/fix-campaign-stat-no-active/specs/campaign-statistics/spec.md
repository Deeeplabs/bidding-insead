## ADDED Requirements

### Requirement: Campaign list stats use 3-tier calculation

Stats must follow this priority:

| Tier | Condition | Stats Source |
|------|-----------|-------------|
| **1** | Pre-bidding OR bidding_round phase is currently active (`start_date <= now <= end_date`) | Aggregate across ALL active pre_bidding + bidding_round phases |
| **2** | No pre_bidding/bidding_round active, but a downstream module (simulation, add_drop_waitlist, etc.) IS active | Fall back to the **closest past bidding round** |
| **3** | No phase of any type is currently active | Return `null` for all stats |

### Requirement: Total students must respect student_filters (include/exclude)

The `total_students` count on the campaign list must apply the same student filtering logic as the preview page:

- **DB table filters** (`campaign_student_filter`): campaign-level + phase-level, scoped by `$phaseConfigId`
- **Config-level filters** (`moduleConfig.config.student_filters`): include/exclude by student_id, student_type, campus, etc.
- **Student selection** (`moduleConfig.student_selection`): `include_all_students` / `include_filtered_students_only`
- **Selected student IDs** (`moduleConfig.config.selected_student_ids`): explicit ID restriction

Both Tier 1 and Tier 2 must pass the **complete config** (including `student_filters` and `student_selection`) along with the correct `phaseConfigId` to `getEligibleStudents()`.

---

#### Scenario: Campaign with active pre-bidding shows stats from that module
- **GIVEN** a campaign with an active pre-bidding phase (30 eligible students, 5 completed)
- **WHEN** the campaign list endpoint is called
- **THEN** `total_students` = 30, `bidding_completed_count` = 5, `bidding_pending_count` = 25

#### Scenario: Campaign with active bidding round shows stats from that module
- **GIVEN** a campaign with an active bidding_round phase (30 eligible students, 10 completed)
- **WHEN** the campaign list endpoint is called
- **THEN** `total_students` = 30, `bidding_completed_count` = 10, `bidding_pending_count` = 20

#### Scenario: Both pre-bidding AND bidding round active → AGGREGATED stats
- **GIVEN** a campaign with both pre-bidding (30 students) and bidding_round (30 students) active
- **WHEN** the campaign list endpoint is called
- **THEN** `total_students` = 60 (30 + 30)
- **AND** bidding completed/pending aggregated across both

#### Scenario: Active bidding with student_include filters → total reflects filtered count
- **GIVEN** a campaign with an active bidding_round phase
- **AND** `moduleConfig.config.student_filters` has `student_include` entries (e.g., 469 specific students)
- **WHEN** the campaign list endpoint is called
- **THEN** `total_students` reflects the filtered count (469), NOT all promotion students
- **AND** this matches the "Eligible Students" count shown on the preview page

#### Scenario: Active bidding with student_exclude filters → total excludes filtered students
- **GIVEN** a campaign with an active bidding_round phase
- **AND** `moduleConfig.config.student_filters` has `student_exclude` entries
- **WHEN** the campaign list endpoint is called
- **THEN** `total_students` excludes those students from the count

#### Scenario: Active bidding with DB-level phase filters → total uses phase-specific filters
- **GIVEN** a campaign with phase-specific `campaign_student_filter` entries (linked to a `phaseConfigId`)
- **WHEN** the campaign list endpoint is called
- **THEN** the stats calculation passes the correct `phaseConfigId` to `getEligibleStudents()`
- **AND** only filters matching that phase (or campaign-level) are applied

#### Scenario: Bidding round completed, add/drop & waitlist active → use past bidding round stats (TIER 2)
- **GIVEN** a campaign where:
  - Pre-Bidding: completed ✅
  - Bidding Round: completed ✅ (e.g., 33 students, 14 completed)
  - Simulation: completed ✅
  - Add/Drop & Waitlist: ACTIVE 🟢 (`start_date <= now <= end_date`)
- **WHEN** the campaign list endpoint is called
- **THEN** stats are calculated from the **completed bidding round** data
- **AND** `total_students` = 33, `bidding_completed_count` = 14, `bidding_pending_count` = 19
- **NOTE**: NOT 0/0/0 — the bidding data is still relevant while downstream modules are active

#### Scenario: Bidding round completed, simulation active → use past bidding round stats (TIER 2)
- **GIVEN** a campaign where:
  - Bidding Round: completed ✅
  - Simulation: ACTIVE 🟢
- **WHEN** the campaign list endpoint is called
- **THEN** stats are calculated from the completed bidding round

#### Scenario: ALL phases completed → null stats (TIER 3)
- **GIVEN** a campaign where ALL phase configs have `end_date` < now
- **AND** the campaign has a Final Enrollment module
- **WHEN** the campaign list endpoint is called
- **THEN** all three stat fields are `null`
- **NOTE**: `final_enrollment` fallback must NOT trigger stats

#### Scenario: Future-dated campaign → null stats (TIER 3)
- **GIVEN** a campaign where all phase configs have `start_date` > now
- **WHEN** the campaign list endpoint is called
- **THEN** all three stat fields are `null`

#### Scenario: Draft campaign → null stats (TIER 3)
- **GIVEN** a campaign with no phase configs
- **WHEN** the campaign list endpoint is called
- **THEN** all three stat fields are `null`

---

### API Response Shape

**Tier 1 — Active pre_bidding/bidding_round (aggregated, with filters applied):**
```json
{
  "total_students": 938,
  "bidding_completed_count": 15,
  "bidding_pending_count": 923
}
```

**Tier 2 — Non-bidding module active, fallback to past bidding round:**
```json
{
  "total_students": 469,
  "bidding_completed_count": 14,
  "bidding_pending_count": 455
}
```

**Tier 3 — Nothing active:**
```json
{
  "total_students": null,
  "bidding_completed_count": null,
  "bidding_pending_count": null
}
```
