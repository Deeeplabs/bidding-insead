## 1. Add PM Notification on Cancel

- [x] 1.1 **Add notification trigger in `FlexSwitchService::cancelRequest()`**
  - File: `bidding-api/src/Service/FlexSwitch/FlexSwitchService.php`
  - In the `cancelRequest()` method (line ~658, after `$this->requestRepository->save($request, true)`)
  - Add notification logic following the same pattern as `submitRequest()` lines 807-835:
    1. Get the student's promotion → program
    2. Find ProgramManagers for the program via `$this->entityManager->getRepository(\App\Entity\ProgramManager::class)->findBy(['program' => $program])`
    3. Collect PM users (`$pm->getUser()`)
    4. Call `$this->notificationService->createBulk()` with:
       - `typeCode: 'CUSTOM_ANNOUNCEMENT'`
       - `recipients: $pmRecipients`
       - `data`:
         - `announcement_title` → `'Flex Switch Request Cancelled'`
         - `announcement_body` → `"Student {firstName} {lastName} has cancelled their switch request from {fromCourseName} to {toCourseName}. Reason: {cancellationReason}."`
         - `request_id` → `$request->getId()`
         - `student_id` → `$student->getId()`
         - `from_course` → from course name (resolve via `$this->entityManager->getRepository(Classes::class)->find($request->getFromClassId())`)
         - `to_course` → to course name (resolve similarly)
         - `cancellation_reason` → `$cancellationReason`
  - Wrap in null-safe checks: `if ($promotion) { if ($program) { ... } }`

## 2. Verification

- [x] 2.1 **Manual verification via API**
  - Call `PUT /v2/api/student/flex-switch/my-requests/{id}/cancel` with a valid cancellation reason on a pending request
  - Verify the request is cancelled successfully (status 200)
  - Check the `notification` table in the database for new PM notification records
  - Verify the notification contains correct title, body, and data fields
