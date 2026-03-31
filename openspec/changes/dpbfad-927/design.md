## Context

The Student Dashboard currently displays a calendar view containing all GEMBA modules without any filtering. This means students see modules for programmes and campuses they are not enrolled in, which causes confusion and clutters the interface. Additionally, FLEX campus modules (virtual/hybrid delivery) should always be visible to all students regardless of their physical home campus.

## Goals / Non-Goals

**Goals:**
- Apply strict backend filtering so the Student Dashboard Calendar automatically returns only modules relevant to the student's programme and allowed campuses.
- Always include FLEX campus (short_name = "FLEX") modules in the calendar for all students, so flex/virtual delivery options are never hidden.
- Ensure no frontend modifications are required to achieve this restricted view.

**Non-Goals:**
- Changing admin-facing UI or admin control of module assignments.
- Altering the Switch Request logic or endpoints (which operate independently and handle full module fetching).

## Decisions

1. **Filtering Strategy:**
   - We will implement pure backend filtering directly in the `/v2/api/student/flex-switch/calendar` endpoint that handles the dashboard calendar data.
   - The backend will strictly filter results to match the allowed campus set and programme before returning them.
   - The Switch Request dropdown uses a separate data context/endpoint to fetch its target module list, so restricting the dashboard response will safely not impact switch functionality.

2. **Campus Allowlist Construction:**
   - The allowed campus set for a student consists of:
     1. The student's **home campus** (`Student::getCampus()`)
     2. Any **exchange campuses** from `Exchange` records (`getP3Campus()`, `getP4Campus()`, `getP5Campus()`)
     3. The **FLEX campus** — looked up from the `Campus` entity where `shortName = 'FLEX'`
   - This ensures students always see flex/virtual delivery modules alongside their location-specific modules.

3. **FLEX Campus Lookup:**
   - The CampusRepository will be used to find the FLEX campus by its `shortName` field.
   - If no campus with `shortName = 'FLEX'` exists in the database, the filter gracefully proceeds without adding it (no error, just no FLEX inclusion).

4. **Identification of Student Context:**
   - The filtering logic will evaluate the `Student` entity (and associated `Promotion` or `Exchange` details if applicable) to determine valid campuses and match them against the dataset.

## Risks / Trade-offs

- [Risk] Aggressive filtering might inadvertently hide valid modules if a student's `Exchange` details or custom program structure isn't properly evaluated.
  - Mitigation: Ensure the filtering logic correctly accounts for the student's primary home campus, any valid active exchange campus assignments, and the FLEX campus.
- [Risk] If the FLEX campus `shortName` value changes in the database, the lookup would silently stop including FLEX modules.
  - Mitigation: The FLEX campus short_name is a well-established constant (used in admin filter UI). If not found, the filter degrades gracefully.
