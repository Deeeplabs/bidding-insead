## Why

The Flex Switch Class Configuration needs more accurate filtering to ensure students are only switching into valid courses, specifically based on Programme, Promotion, and a "Core" course type. The frontend also requires `module` and `term` details in the API response. Additionally, validations must be added to prevent students from switching to courses they are already enrolled in or courses with schedule conflicts. Finally, bid points rule configuration must be checked and enforced. This solves filtering gaps and schedule conflict issues during flex switching.

## What Changes

- **Filter Updates**: Ensure `/v2/api/flex-switch-class-configuration` filters courses correctly by Programme, Promotion, and type: Core.
- **API Response**: Add `module` and `term` columns to the `/v2/api/flex-switch-class-configuration` API response.
- **Validations**: Do not allow students to take courses they have already enrolled in, or courses that have direct schedule conflicts.
- **Rules Verification**: Check and ensure Rules Configuration (specifically Bid Points) are fully implemented for this workflow.
- **Notification Triggers**: Add notification triggers to the `POST dashboard/flex-switch/approval-requests/{id}/process` and `POST student/flex-switch/request` endpoints.

## Capabilities

### New Capabilities
- `flex-switch-validations`: New capability for flex-switch specifically validating schedule conflicts and prior enrollments before allowing a switch.

### Modified Capabilities
- `flex-switch`: The flex-switch configuration API is being updated with stricter filtering (promotion, programme, core type) and enriched response data (module, term).

## Impact

- **API Endpoint (`/v2/api/flex-switch-class-configuration`)**: Output will be modified to include `module` and `term`. Filtering logic in the Domain Service / Repository will be updated.
- **Validation Logic**: Flex Switch requests / course allocations will require new checks against existing enrollments and course conflict services.
- **Controllers & Services**: Modification to the Controller or Dto/Transformer for the flex-switch configuration response.
- No existing fields removed, so no breaking API changes, purely additions and stricter query filtering.
