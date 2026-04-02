## 1. Static Seat Display in Student Portal (bidding-web)

- [x] 1.1 Update `CourseOptionContent` component in `bidding-web/src/features/bidding/components/bid-submission/CourseOptionContent.tsx` to replace the dynamic seat chip. Change the chip from `{data.available_seats}/{data.total_seats} seats` to `{data.total_seats} seats`. Remove the conditional green/red coloring based on `data.available_seats > 0`. Apply a single neutral chip style (e.g., `!border !bg-white !text-gray-700 !border-gray-300`). This single change applies to both Bidding Round and Add/Drop since both phases use the same `CourseOptionContent` component.

- [x] 1.2 Verify that no other student-facing components in `bidding-web/src/features/bidding/` render `available_seats` or `enrolled_count` to the user. Check `CourseTable.tsx`, `AddDropPage.tsx`, and any bidding round page components. If any display dynamic seat data, update them to show only `total_seats`.

## 2. Fix PM Dashboard Seat Format (bidding-api)

- [x] 2.1 Add an `is_shared` boolean property to `AdminBiddingRoundCourseDto` in `bidding-api/src/Domain/Dashboard/Campaign/AdminBiddingRoundCourseDto.php`. Add the corresponding `#[OA\Property]` annotation. This field indicates whether the course is shared/split across multiple programme-promotions.

- [x] 2.2 Update `AdminCampaignDetailService::buildBiddingRoundDetail()` in `bidding-api/src/Domain/Dashboard/Campaign/AdminCampaignDetailService.php` to compute and set `is_shared` on each `AdminBiddingRoundCourseDto`. Determine if the class has `ClassPromotions` records associated with more than one distinct promotion. Set `$courseDto->is_shared = count(distinct promotions in ClassPromotions) > 1`.

- [x] 2.3 Write PHPUnit tests in `bidding-api/tests/Unit/Domain/Dashboard/Campaign/` to verify: (a) `is_shared = false` when a class has ClassPromotions for only 1 promotion, (b) `is_shared = true` when a class has ClassPromotions for multiple promotions, (c) `total_seats` and `total_all_seats` remain correctly computed.

## 3. Fix PM Dashboard Seat Column (bidding-admin)

- [x] 3.1 Update `course-table-bidding-round.tsx` in `bidding-admin/src/components/dashboard/process/bidding-round/` to change the seat column render logic. Replace the current condition `row.total_all_seats && row.total_all_seats !== row.total_seats` with `row.is_shared === true`. When `is_shared` is true, display `{row.total_seats} ({row.total_all_seats})`. When false, display only `{row.total_seats}`.

- [x] 3.2 Update the `CourseListBidding` TypeScript type in `bidding-admin/src/src/campaign-management/campaign.types.ts` to add `is_shared?: boolean` field.

- [x] 3.3 If there is an equivalent Add/Drop phase course table in bidding-admin that also displays seats, apply the same `is_shared`-based format logic there. Check `bidding-admin/src/components/dashboard/process/` for add-drop phase course tables.

## 4. Verify Backend Enrollment Validation (bidding-api)

- [x] 4.1 Verify that `AddDropService::executeCampaignAddDrop()` in `bidding-api/src/Domain/Campaign/ActiveCampaign/AddDropService.php` returns a clear error message (e.g., "Course is full") when seat capacity is reached. If the current error message is generic or unclear, update it to explicitly state the course is full. Ensure the existing pessimistic locking (tasks 2.1/2.2 from prior implementation) is in place.

- [x] 4.2 Write or update PHPUnit tests to verify: (a) enrollment returns ENROLLED when seats are available, (b) enrollment returns a clear "Course is full" error when seats are exhausted, (c) concurrent enrollment attempts do not exceed seat capacity.

## 5. Verification and Smoke Testing

- [ ] 5.1 Manual UI verification (bidding-web, port 4006): During an active Bidding Round campaign, open the course selector and verify each course shows only `XX seats` (static total) — no remaining/available counts, no green/red coloring.

- [ ] 5.2 Manual UI verification (bidding-web, port 4006): During an active Add/Drop campaign, open the course selector and verify each course shows only `XX seats` (static total). Attempt to enroll in a full course and verify a clear "Course is full" error appears.

- [ ] 5.3 Manual UI verification (bidding-admin, port 4007): View PM dashboard bidding round detail. Verify non-shared courses show only `XX` in the seat column. Verify shared courses (split across promotions) show `XX (XX)` format.

- [ ] 5.4 Concurrency smoke test: Open two browser sessions as different students, both targeting the same course with 1 seat remaining. Submit enrollment simultaneously from both. Verify exactly one succeeds and the other gets a "Course is full" error.

- [ ] 5.5 Regression check: Run existing PHPUnit test suite (`bin/phpunit`) to verify no existing tests broken by the changes.
