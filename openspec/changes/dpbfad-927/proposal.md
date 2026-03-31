## Why

Students can currently see all GEMBA modules in the calendar view, including unrelated programmes. This clutters the interface and can confuse students about their upcoming modules. The change aims to provide a focused view tailored precisely to each student.

## What Changes

The backend endpoint responsible for populating the Student Dashboard Calendar View (`/v2/api/student/flex-switch/calendar`) will be updated to automatically filter out modules that do not match the student's current programme and home campus.
No frontend changes are needed, as the frontend will simply render the filtered dataset provided by the backend.

## Capabilities

### New Capabilities

- `dashboard-module-filtering`: Filters the student dashboard layout from the backend to present only programme and home campus relevant modules to the student by default.

### Modified Capabilities


## Impact

- `bidding-api`: The `/v2/api/student/flex-switch/calendar` endpoint needs modification in the module retrieval logic to strictly apply the programme and home campus filter.
- `bidding-web`: No changes expected. The frontend will consume the newly filtered backend response natively.
