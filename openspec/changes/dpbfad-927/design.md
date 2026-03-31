## Context

The Student Dashboard currently displays a calendar view containing all GEMBA modules without any filtering. This means students see modules for programmes and home campuses they are not enrolled in, which causes confusion and clutters the interface. 

## Goals / Non-Goals

**Goals:**
- Apply strict backend filtering so the Student Dashboard Calendar automatically returns only modules relevant to the student's programme and home campus.
- Ensure no frontend modifications are required to achieve this restricted view.

**Non-Goals:**
- Changing admin-facing UI or admin control of module assignments.
- Altering the Switch Request logic or endpoints (which operate independently and handle full module fetching).

## Decisions

1. **Filtering Strategy:**
   - We will implement pure backend filtering directly in the `/v2/api/student/flex-switch/calendar` endpoint that handles the dashboard calendar data. 
   - Based on recent feedback, whatever parameter is passed from the dashboard frontend, the backend will strictly filter results to match the home campus and programme before returning them.
   - The Switch Request dropdown uses a separate data context/endpoint to fetch its target module list, so restricting the dashboard response will safely not impact switch functionality.

2. **Identification of Student Context:**
   - The filtering logic will evaluate the `Student` entity (and associated `Promotion` or `Exchange` details if applicable) to determine valid campuses and match them against the dataset.

## Risks / Trade-offs

- [Risk] Aggressive filtering might inadvertently hide valid modules if a student's `Exchange` details or custom program structure isn't properly evaluated.
  - Mitigation: Ensure the filtering logic correctly accounts for both the student's primary home campus and any valid active campus assignments.
