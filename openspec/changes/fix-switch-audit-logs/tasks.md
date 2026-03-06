## 1. Add Audit Logging to FlexSwitchApprovalService (PM approve/reject)

- [x] 1.1 **Add `AuditLogService` dependency to `FlexSwitchApprovalService`**
  - File: `bidding-api/src/Service/FlexSwitchApprovalService.php`
  - Add `use App\Domain\AuditLog\AuditLogService;` import
  - Add `private AuditLogService $auditLogService` to constructor parameters
  - Follow same pattern as `PM\FlexSwitchService` constructor injection

- [x] 1.2 **Add audit log calls to `processApproval()` method**
  - File: `bidding-api/src/Service/FlexSwitchApprovalService.php`
  - After the `$this->entityManager->flush()` call (line 286), add audit log call:
    - Action: `'APPROVE FlexSwitchRequest'` or `'REJECT FlexSwitchRequest'` based on `$dto->action`
    - Entity type: `'FlexSwitchRequest'`
    - Entity ID: `$request->getId()`
    - Old data: `{ status: 'pending', student_id, from_class_id, to_class_id, from_course_id, to_course_id }`
    - New data: `{ status: approved/rejected, approved_by: approver_id, approved_at, remarks }`
    - Description: Human-readable summary like "FlexSwitch request #42 approved by PM (email) for student (name)"
  - Follow the pattern from `PM\FlexSwitchService.createConfiguration()` (lines 770-786)

## 2. Add Audit Logging to FlexSwitchService - Student Submit (student side)

- [x] 2.1 **Add `AuditLogService` dependency to `FlexSwitchService` (student)**
  - File: `bidding-api/src/Service/FlexSwitch/FlexSwitchService.php`
  - Add `use App\Domain\AuditLog\AuditLogService;` import
  - Add `private readonly AuditLogService $auditLogService` to constructor parameters

- [x] 2.2 **Add audit log call to `submitRequest()` method**
  - File: `bidding-api/src/Service/FlexSwitch/FlexSwitchService.php`
  - After the flush that persists the new `FlexSwitchRequest`, add audit log call:
    - Action: `'CREATE FlexSwitchRequest'`
    - Entity type: `'FlexSwitchRequest'`
    - Entity ID: `$request->getId()`
    - Old data: `null`
    - New data: `{ student_id, from_class_id, to_class_id, from_course_id, to_course_id, reason, status: 'pending' }`
    - Description: "Student (name) submitted FlexSwitch request to switch from (course) to (course)"

## 3. Add Audit Logging to FlexSwitchService - Student Cancel (student side)

- [x] 3.1 **Add audit log call to `cancelRequest()` method**
  - File: `bidding-api/src/Service/FlexSwitch/FlexSwitchService.php`
  - After the flush that updates the request status to cancelled, add audit log call:
    - Action: `'CANCEL FlexSwitchRequest'`
    - Entity type: `'FlexSwitchRequest'`
    - Entity ID: `$requestEntity->getId()` (or `$requestId`)
    - Old data: `{ status: 'pending' }`
    - New data: `{ status: 'cancelled', cancelled_by: student_id, cancelled_at, cancellation_reason }`
    - Description: "Student (name) cancelled FlexSwitch request #(id)"

## 4. Manual Verification

- [x] 4.1 **Verify audit logs appear for PM approval/rejection**
  - Login as PM, go to Flex Switch, approve or reject a request
  - Go to Monitoring & Analytics > Audit Logs
  - Confirm audit log entries appear with action `APPROVE FlexSwitchRequest` or `REJECT FlexSwitchRequest`
  - Verify old_data and new_data contain correct status transitions

- [x] 4.2 **Verify audit logs appear for student submit/cancel**
  - Login as student, submit a new flex switch request
  - Cancel a pending flex switch request
  - Go to Audit Logs (as admin) and confirm entries for `CREATE FlexSwitchRequest` and `CANCEL FlexSwitchRequest`
