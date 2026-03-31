## Why

Students can currently see all GEMBA modules in the calendar view, including unrelated programmes and campuses. This clutters the interface and can confuse students about their upcoming modules. The change aims to provide a focused view tailored precisely to each student's programme, home campus, exchange campuses, and any FLEX (virtual/hybrid delivery) campus modules.

## What Changes

The backend endpoint responsible for populating the Student Dashboard Calendar View (`/v2/api/student/flex-switch/calendar`) will be updated to automatically filter out modules that do not match the student's current programme and allowed campuses. The allowed campus set includes:
- The student's home campus
- Any campuses assigned via Exchange records (P3, P4, P5)
- The **FLEX** campus (short_name = "FLEX"), so students always see flex/virtual delivery modules regardless of their physical home campus

No frontend changes are needed, as the frontend will simply render the filtered dataset provided by the backend.

## Capabilities

### New Capabilities

- `dashboard-module-filtering`: Filters the student dashboard calendar from the backend to present only programme-relevant and campus-relevant modules to the student by default, including FLEX campus modules for all students.

### Modified Capabilities


## Impact

- `bidding-api`: The `/v2/api/student/flex-switch/calendar` endpoint needs modification in the module retrieval logic to apply the programme filter and extended campus filter (home + exchange + FLEX).
- `bidding-web`: No changes expected. The frontend will consume the newly filtered backend response natively.
