# Fix Edit Switch Validation and API Enrichment

## Problem
The `fix-edit-switch` feature required stricter filtering, enriched API responses, comprehensive domain validations, and notification triggers to prevent invalid student enrollment switches and keep students informed. The original implementation was missing critical rule evaluations, suffered from architecture divergence, and lacked automated communication.

1. **Filtering & Payload Weakness**: The `/v2/api/flex-switch-class-configuration` endpoint did not strictly filter by the selected Programme, Promotion, or Core `courseType`. Additionally, essential `module` and `term` details were missing from the response payloads.
2. **Missing Rule Validations**: Bid points configurations mapped by the PM natively to `FlexSwitchConfiguration` were completely ignored during flex switch submissions, allowing students to switch regardless of campaign point constraints.
3. **Architecture Adherence**: Validations for matching "Already Enrolled" courses and identifying "Schedule Conflicts" were built generically in the `FlexSwitchController` layer as basic Doctrine Queries rather than being encapsulated within the domain components, creating business logic leakage.
4. **Missing Notifications**: Students were not notified when they explicitly submitted a flex switch request or when the Programme Manager subsequently processed (approved/rejected) their request, resulting in a disconnected user experience.

## Solution
Centralized flex switch validations natively into the Domain services, tightened entity resolution, enforced PM rule configurations, enriched the API response payloads safely, and integrated automated notification triggers.

### Changes Made

**Modified Files:**

1. **`src/Service/FlexSwitch/PM/FlexSwitchService.php`**
   - Modified `listClassConfiguration()` to strictly enforce filtering against `program_id` and `promotion_id` parameters passed in mapping to the promotion constraints.
   - Updated the data transfer query (DQL) selecting the promotion period's `term` metadata utilizing `MAX(pp.term)` property mapping.

2. **`src/Controller/Api/FlexSwitch/FlexSwitchController.php`**
   - Successfully bound request parsing logic to strictly inject `program_id` and `promotion_id` parameters from `$request->query` for consumption by `PM\FlexSwitchService::listClassConfiguration()`.

3. **`src/Domain/FlexSwitch/FlexSwitchClassConfigurationItemDto.php`**
   - Appended a new optional string attribute for `$term` properties supporting API structure rendering logic.

4. **`src/Service/FlexSwitch/FlexSwitchService.php`**
   - Implemented standard `submitRequest()` domain logic entirely encapsulating validation workflows.
   - Migrated "Schedule Conflicts" array checking from DQL logic deep into the structured domain block parsing target class constraints mapping actively enrolled objects seamlessly.
   - Migrated "Already Enrolled" overlap DQL checks cleanly into the domain layer enforcing the swap operation accurately queries the distinct Target node rejecting duplicate core enrollments natively.
   - Enforced **FlexSwitch Rule Configuration** queries mapping the student's current `Promotion` against `rules.deduct_points` and `rules.minimum_points`. Blocks switches throwing `\DomainException` if the capital threshold falls below minimum.
   - Injected the `NotificationService` to send a `CUSTOM_ANNOUNCEMENT` notification back to the student upon successful flex switch request submission.

5. **`src/Controller/Api/Student/FlexSwitch/FlexSwitchController.php`**
   - Purged all leaked Doctrine Queries and generic validations out of the Web boundary `submitRequest()`.
   - Replaced extensive validation logic with a single call to `flexSwitchService->submitRequest()`, cleanly mapping isolated `\DomainException`s back natively to the expected HTTP 400 JSON standard (`INVALID_CLASS`, `ALREADY_ENROLLED`, `SCHEDULE_CONFLICT`, `INSUFFICIENT_POINTS`).

6. **`src/Service/FlexSwitchApprovalService.php`**
   - Injected the `NotificationService` and securely hooked it into `processApproval()` to trigger a `CUSTOM_ANNOUNCEMENT` notification to the student outlining the request action (approved/rejected) and any PM remarks.

## Impact
- **Data Completeness:** API consumers strictly receive `module`, `term` and `location` details properly correlated to PM promotion periods for Class matching.
- **System Integrity:** Flex switch backend strictly filters available swapping combinations dynamically matching the student's configured Promotion and restricted to core courses.
- **Application Coherence:** Business rules validating target overlaps, schedule conflicts, and active rule configurations (capital point validations) are reliably processed and locked securely at the correct domain service abstraction layer eliminating bypassing flaws.
- **Improved User Experience:** Students receive immediate system notifications verifying their flex switch request submissions and the final approval/rejection resolution directly from the Programme Manager.
