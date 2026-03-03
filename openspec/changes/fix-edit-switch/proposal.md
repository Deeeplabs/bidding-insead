## Why

The Flex Switch Class Configuration needs more accurate filtering to ensure students are only switching into valid courses, specifically based on Programme, Promotion, and a "Core" course type. The frontend also requires `module` and `term` details in the API response. Additionally, validations must be added to prevent students from switching to courses they are already enrolled in or courses with schedule conflicts. Finally, bid points rule configuration must be checked and enforced. The search functionality also needs to be enhanced to allow searching by module name and home campus. This solves filtering gaps and schedule conflict issues during flex switching.

## What Changes

- **Filter Updates**: Ensure `/v2/api/flex-switch-class-configuration` filters courses correctly by Programme, Promotion, and type: Core.
- **API Response**: Add `module` and `term` columns to the `/v2/api/flex-switch-class-configuration` API response.
- **Validations**: Do not allow students to take courses they have already enrolled in, or courses that have direct schedule conflicts. Enforce `max_submissions_allowed_per_student`.
- **Rules Verification**: Check and ensure Rules Configuration (specifically Bid Points and Max Submissions per student) are fully implemented for this workflow.
- **Notification Triggers**: Add notification triggers to the `POST dashboard/flex-switch/approval-requests/{id}/process` and `POST student/flex-switch/request` endpoints, ensuring content correctly replaces `{{announcement_title}}` and `{{announcement_body}}`.
- **Enhanced Search**: Add support for searching by module name and campus (home campus) in the `/student/flex-switch/switch-to-courses` endpoint, in addition to existing course name, course ID, and section searches.

## Capabilities

### New Capabilities
- `flex-switch-validations`: New capability for flex-switch specifically validating schedule conflicts and prior enrollments before allowing a switch.
- `flex-switch-search`: Enhanced search capability allowing students to search for switch target courses by module name or campus.

### Modified Capabilities
- `flex-switch`: The flex-switch configuration API is being updated with stricter filtering (promotion, programme, core type) and enriched response data (module, term). The switch-to-courses endpoint now supports enhanced search by module and campus.

## Impact

- **API Endpoint (`/v2/api/flex-switch-class-configuration`)**: Output will be modified to include `module` and `term`. Filtering logic in the Domain Service / Repository will be updated.
- **API Endpoint (`/student/flex-switch/switch-to-courses`)**: The `search` query parameter now supports searching by module name (e.g., "Module 1") and campus name/code (e.g., "Singapore", "SGP"), in addition to existing course name, course ID, and section.
- **Validation Logic**: Flex Switch requests / course allocations will require new checks against existing enrollments and course conflict services.
- **Controllers & Services**: Modification to the Controller or Dto/Transformer for the flex-switch configuration response.
- No existing fields removed, so no breaking API changes, purely additions and stricter query filtering.
