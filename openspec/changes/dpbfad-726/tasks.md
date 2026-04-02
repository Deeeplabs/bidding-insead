## 1. Fix Available Course Filtering (bidding-api)

- [x] 1.1 Update `AddDropAvailableCourseService::getAvailableCourses()` in `bidding-api/src/Domain/Campaign/ActiveCampaign/AddDrop/AddDropAvailableCourseService.php` to add fallback seat calculation when `ClassPromotions` records are missing or promotion matching fails. When `ClassPromotions`-based seats return zero, check `SimulationAdjustmentCourse` and `AdjustmentCourse` overrides before excluding the course. Log a warning via Symfony logger when falling back to override-based seats.

- [x] 1.2 Update `CampaignCourseService::buildAvailableCourse()` in `bidding-api/src/Domain/Campaign/ActiveCampaign/CampaignCourseService.php` to relax the non-zero `ClassPromotions.promotionSeats` filter. Allow courses through when `SimulationAdjustmentCourse` or `AdjustmentCourse` seat overrides exist with capacity > 0, even if ClassPromotions-based calculation returns zero.

- [ ] 1.3 Write PHPUnit tests in `bidding-api/tests/Unit/Domain/Campaign/ActiveCampaign/AddDrop/` to verify: (a) courses with seats via ClassPromotions are returned, (b) courses with seats via SimulationAdjustmentCourse override are returned when ClassPromotions missing, (c) courses with seats via AdjustmentCourse override are returned as fallback, (d) courses with zero capacity from all sources are excluded.

## 2. Add Pessimistic Locking for Seat Allocation (bidding-api)

- [x] 2.1 Add a locking query method to `BidRepository` (`bidding-api/src/Repository/BidRepository.php`) that uses `LockMode::PESSIMISTIC_WRITE` when counting enrolled bids for a class within a campaign. This method will be called during enrollment to prevent concurrent reads.

- [x] 2.2 Update `AddDropService::executeCampaignAddDrop()` in `bidding-api/src/Domain/Campaign/ActiveCampaign/AddDropService.php` to wrap the seat availability check and bid creation within an explicit Doctrine transaction. Use the new locking repository method from task 2.1 to count enrolled bids before creating the enrollment bid.

- [ ] 2.3 Write PHPUnit tests to verify: (a) enrollment succeeds and returns ENROLLED when seats are available, (b) enrollment returns WAITLISTED when seats are full, (c) the locking query method is called during enrollment flow.

## 3. Add Seat Availability Fields to Response DTO (bidding-api)

- [x] 3.1 Add `available_seats` (int), `total_seats` (int), and `enrolled_count` (int) fields to the available course response DTO in `bidding-api/src/Dto/Response/`. Ensure these are additive fields — no existing fields removed or renamed. Update the corresponding Response Transformer to populate these fields from `AddDropAvailableCourseService` data.

- [x] 3.2 Verify existing response fields remain unchanged by running existing API tests. Update OpenAPI annotations (`#[OA\...]`) on the available courses controller to document the new fields.

## 4. Add Sort and Filter Query Parameters (bidding-api)

- [x] 4.1 Add optional `availability` query parameter (`available` | `all`, default `all`) and `sort_by` query parameter (`available_seats_desc` | `course_name_asc`, default existing order) to `AddDropAvailableCourseController` in `bidding-api/src/Controller/Api/Student/Campaign/AddDrop/AddDropAvailableCourseController.php`. Pass these parameters through to `AddDropAvailableCourseService`.

- [x] 4.2 Update `AddDropAvailableCourseService::getAvailableCourses()` to apply server-side filtering (exclude full courses when `availability=available`) and sorting (order by available seats descending when `sort_by=available_seats_desc`) before pagination.

- [ ] 4.3 Write PHPUnit tests to verify: (a) `availability=available` excludes full courses, (b) `availability=all` returns all courses, (c) `sort_by=available_seats_desc` orders correctly, (d) default parameters preserve existing behavior.

## 5. Update Student Portal UI (bidding-web)

- [x] 5.1 Update the add-drop available course TypeScript types in `bidding-web/src/features/bidding/types/add-drop.type.ts` to include `available_seats`, `total_seats`, and `enrolled_count` fields in the available course type.

- [x] 5.2 Update `CourseSelector` component in `bidding-web/src/features/bidding/components/bid-submission/CourseSelector.tsx` to display seat availability (e.g., "3/30 seats available") next to each course-section option. Highlight sections with available seats vs full sections visually.

- [x] 5.3 Add availability filter and sort controls to the add/drop course selection UI. Add a toggle/filter to show only courses with available seats and a sort option for "most seats available". Update the API service call in `bidding-web/src/lib/api/services/campaign.service.ts` and query in `bidding-web/src/lib/api/queries/campaign.queries.ts` to pass the new `availability` and `sort_by` parameters.

- [x] 5.4 Update `use-add-drop-waitlist-form.tsx` in `bidding-web/src/features/bidding/hooks/` to invalidate the `addDropAvailableCourses` query key and the student enrollment list query after successful enrollment submission, so seat counts and enrollment status refresh automatically.

## 6. Verification and Smoke Testing

- [ ] 6.1 Manual API verification: Using curl or API docs (`/api/doc`), test the available courses endpoint with various parameter combinations (`availability=available`, `sort_by=available_seats_desc`) and verify response includes `available_seats`, `total_seats`, `enrolled_count` fields.

- [ ] 6.2 Manual UI verification: In bidding-web (port 4006), navigate to add/drop page during an active campaign, verify: (a) courses with available seats appear with seat counts, (b) filter to show only available courses works, (c) enrolling in a course shows updated seat count without manual refresh, (d) full courses show as full with option to join waitlist.

- [ ] 6.3 Concurrency smoke test: Open two browser sessions as different students, both viewing the same course with 1 seat. Submit enrollment simultaneously from both. Verify exactly one gets ENROLLED and the other gets WAITLISTED.

- [ ] 6.4 Regression check: Run existing PHPUnit test suite (`bin/phpunit`) to verify no existing tests broken by the changes.
