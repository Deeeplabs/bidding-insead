## 1. Validations Implementation

- [x] 1.1 Locate the Flex Switch Request processing service (e.g., in `src/Domain/FlexSwitch/Service/`).
- [x] 1.2 Implement validation to check if the student is already enrolled in the requested switch target course.
- [x] 1.3 Implement validation to query the schedule / class timing of the target course against the student's active enrolled classes, returning a conflict error if there is overlap.
- [x] 1.4 Make sure these validations throw standard domain validation exceptions.

## 2. Rule Configuration Checks

- [x] 2.1 Locate the existing generic rules validation service or bid points rules service (e.g., `BiddingRuleService` or `CampaignRuleService`).
- [x] 2.2 Re-use or inject this rule configuration checker into the FlexSwitch process.
- [x] 2.3 Verify bid points validation runs accurately. No points are subtracted if rules don't permit it, or requirements are enforced per the PM's campaign rules configuration.
- [x] 2.4 Query the student's existing flex switch submission count for the active context.
- [x] 2.5 Add validation on `/v2/api/student/flex-switch/request` to reject the request if the student has reached the `max_submissions_allowed_per_student` limit configured in rules.

## 3. Query & Output Payload Enrichment

- [x] 3.1 Update the Repository / Query logic for `/v2/api/flex-switch-class-configuration` to strictly filter by `Programme`, `Promotion`, and `CampaignModule` `type = 'Core'`.
- [x] 3.2 Add `module` and `term` (or associated string descriptors) to the Response DTO (e.g., `FlexSwitchClassConfigurationResponse`).
- [x] 3.3 Update the corresponding Transformer corresponding to the Response DTO so it correctly maps the `module` and `term` entity fields into the DTO properties.

## 4. Verification

- [x] 4.1 Perform a manual smoke test hitting the `/v2/api/flex-switch-class-configuration` API endpoint to confirm backwards compatibility and the presence of new fields.

## 5. Notification Triggers

- [x] 5.1 Locate the endpoint `POST student/flex-switch/request` and its corresponding service.
- [x] 5.2 Implement logic to trigger a notification to the student when a flex switch request is submitted.
- [x] 5.3 Implement logic to trigger a notification to the Programme Managers (PMs) of the student's program when a flex switch request is submitted.
- [x] 5.4 Locate the endpoint `POST dashboard/flex-switch/approval-requests/{id}/process` and its corresponding service.
- [x] 5.5 Implement logic to trigger a notification when the dashboard processes a flex switch approval request.
- [x] 5.6 Ensure the parameters passed to the notification properly resolve the variables `{{announcement_title}}` and `{{announcement_body}}` to actual formatted content instead of raw template placeholders.

## 6. Enhanced Search Functionality

- [x] 6.1 Locate the `getSwitchToCourses()` method in `FlexSwitchService.php`.
- [x] 6.2 Update the search query to include module name matching using CONCAT expressions (format: "Module X: DELIVERY_MODE CAMPUS").
- [x] 6.3 Add campus name and campus code to the search criteria.
- [x] 6.4 Update OpenAPI documentation in `FlexSwitchController.php` to reflect the enhanced search parameter description.
- [x] 6.5 Test the search functionality with module and campus search terms (e.g., `?search=module`, `?search=Singapore`, `?search=SGP`).

## 7. Campus Exclusion Filter

- [x] 7.1 Locate the query logic that fetches switch target classes (e.g., `getSwitchToCourses()` in `FlexSwitchService.php` or its repository).
- [x] 7.2 Implement a filter to exclude target classes that share the same campus as the student's current campus.
- [x] 7.3 Verify that if a student is from the Abu Dhabi (ABD) campus, classes located in ABD do not appear in the available switch options.

## 8. Fix Class Configuration Filters (400 Errors)

- [x] 8.1 Fix HAVING clause bug in `FlexSwitchService::listClassConfiguration()`: replace `having()` with `andHaving()` for the second seat condition so both `seat_min` and `seat_max` work together. The current code uses `having()` which replaces the previous HAVING clause, leaving an orphaned `:seatMin` parameter that causes a Doctrine `QueryException`.
- [x] 8.2 Fix parameter name mismatch in `FlexSwitchController::listClassConfiguration()`: the controller reads `program_id` / `promotion_id` but the frontend sends `programmeId` / `promotionId`. Add fallback to accept both naming conventions.
- [ ] 8.3 Update `useListClassConfiguration` hook in `use-flex-switch.ts` to use a proper type that includes all filter fields (`campus_ids`, `seat_min`, `seat_max`, `credit_min`, `credit_max`, `modes`) instead of the restrictive `ParamsApprovalSwitch` type.
- [x] 8.4 Fix duplicate DQL alias `cp3` in the count query: when both `seat_min` and `seat_max` are set, the same subquery builder reused alias `cp3` for both conditions, causing Doctrine `QueryException: 'cp3' is already defined`. Fix: use separate aliases `cp3` (min) and `cp4` (max).
- [x] 8.5 Fix Symfony `InputBag::get()` for array parameters: `campus_ids` and `modes` are sent as `campus_ids[]=5` (arrays), but `$request->query->get()` throws `BadRequestException` for non-scalar values in Symfony 5.1+/6+. Fix: use `$request->query->all('campus_ids')` and `$request->query->all('modes')`.
- [ ] 8.6 Verify the `/v2/api/flex-switch-class-configuration` endpoint with campus and seat filters returns 200 with correct data.
