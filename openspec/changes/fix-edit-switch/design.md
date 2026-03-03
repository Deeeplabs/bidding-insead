## Context

The Flex Switch feature allows students to switch their enrolled courses. Currently, the `/v2/api/flex-switch-class-configuration` endpoint does not enforce strict filtering by Programme and Promotion to limit courses only to "Core" types. Additionally, the response lacks `module` and `term` details which are necessary for the frontend. The current system also lacks robust validation to prevent a student from switching into a course they are already enrolled in, or one that has schedule conflicts. Bid point rules configuration must also be verified to ensure proper deduction or validation during the switch. 

## Goals / Non-Goals

**Goals:**
- Update `/v2/api/flex-switch-class-configuration` to filter for Core courses matching the student's Programme and Promotion.
- Enrich the same API endpoint's response DTO to include `module` and `term`.
- Implement validation checks against existing student enrollments and schedule conflicts during a flex switch request.
- Ensure bid points rule configurations are correctly checked and applied.
- Trigger notifications upon submission of a request (`POST student/flex-switch/request`) and processing of an approval request (`POST dashboard/flex-switch/approval-requests/{id}/process`).

**Non-Goals:**
- Completely overhauling the simulation or allocation engine.
- Changing the structure of non-flex-switch endpoints.
- Introducing new core Domain entities.

## Decisions

1. **API Response Modification (`flex-switch-class-configuration`)**: We will update the corresponding Response DTO (likely `FlexSwitchClassConfigurationResponse`) and its associated Transformer to populate `module` and `term` fields. This is an additive change and maintains backward compatibility.
2. **Filtering Logic**: The query that fetches available configurations for flex-switch will be modified (likely in the Repository or Domain query service) to eagerly filter by `Campaign`'s Programme, Promotion, and ensure the course type is Core.
3. **Conflict Validation**: We will add domain validation logic inside the `FlexSwitchRequest` handling service to:
    - Check the student's existing `Enrollment` / `Bid` state. If already enrolled in the exact target course, throw a validation exception.
    - Check class scheduling to reject conflicting class times.
4. **Bid Points Rule Verification**: We will inject the existing Bid Points rule checker into the Flex Switch workflow to validate that rules configuration is honored during a flex switch.
5. **Notification Triggers**: We will add hooks or listeners to dispatch notifications when a student submits a flex switch request, and when a dashboard PM/admin processes an approval request.

## Risks / Trade-offs

- **Risk: Breaking Frontend Clients** → Mitigation: All fields added to the DTOs are strictly additive. No existing fields will be removed.
- **Risk: Increased Query Complexity** → Mitigation: Use optimal DQL to fetch Programme and Promotion relationships without causing N+1 issues.
- **Risk: Blocking Legitimate Switches** → Mitigation: Ensure conflict checks do not incorrectly flag past (archived) enrollments; only check active sessions or campaigns.
