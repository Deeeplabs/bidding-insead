## Why

During the Add/Drop phase, courses with available seats are not reliably surfaced to students for direct enrollment. The `AddDropAvailableCourseService` applies multiple filters (class status, promotion matching, ClassPromotions existence, seat capacity checks) that can silently exclude courses even when seats remain. Additionally, there is no concurrency protection on seat allocation, risking over-enrollment when multiple students enroll simultaneously. Students cannot easily discover which sections have open seats, and the UI does not provide a clear path to directly enroll in available courses without going through the full bidding/add-drop form workflow.

## What Changes

- **Fix available course filtering**: Ensure `AddDropAvailableCourseService` and `CampaignCourseService` do not silently exclude courses that have available seats due to missing or mismatched `ClassPromotions` records, incorrect promotion matching, or stale campaign course filters.
- **Add real-time seat availability display**: Surface accurate, live seat counts in the student portal's add/drop course list so students can immediately identify sections with open seats.
- **Add concurrency protection for seat allocation**: Implement database-level locking (pessimistic or optimistic) in `AddDropService` to prevent race conditions where multiple students simultaneously claim the last seat, causing over-allocation.
- **Enhance course discoverability**: Add filtering/sorting options in the student UI to allow students to easily find courses with available seats (e.g., sort by availability, filter to show only courses with open seats).
- **Immediate seat count update**: Ensure seat counts decrement immediately after enrollment and the UI reflects the updated availability without requiring a page refresh.

## Capabilities

### New Capabilities
- `add-drop-seat-availability`: Real-time seat availability display, course discoverability filters (sort/filter by available seats), and immediate seat count updates in the student add/drop UI.

### Modified Capabilities
_(No existing specs to modify — the openspec/specs/ directory is empty)_

## Impact

**bidding-api:**
- `Domain/Campaign/ActiveCampaign/AddDrop/AddDropAvailableCourseService.php` — Fix filtering logic that silently excludes available courses (promotion matching, ClassPromotions checks, seat capacity threshold).
- `Domain/Campaign/ActiveCampaign/CampaignCourseService.php` — Review and fix campaign course filter application that may incorrectly hide classes.
- `Domain/Campaign/ActiveCampaign/AddDropService.php` — Add database-level locking around seat allocation to prevent race conditions on concurrent enrollments.
- `Repository/BidRepository.php` — May need locking queries (SELECT ... FOR UPDATE) for seat counting during enrollment.
- `Dto/Response/` — Add/update response DTOs to include real-time seat availability fields (available_seats, total_seats, enrolled_count) in available course responses.
- `Controller/Api/Student/Campaign/AddDrop/AddDropAvailableCourseController.php` — Add sort/filter parameters for seat availability.

**bidding-web:**
- `features/bidding/components/bid-submission/CourseSelector.tsx` — Display seat availability per section, highlight courses with open seats.
- `features/bidding/hooks/use-add-drop-waitlist-form.tsx` — Handle immediate UI updates after enrollment.
- `lib/api/queries/campaign.queries.ts` — Add filter/sort parameters to available courses query.
- `lib/api/services/campaign.service.ts` — Update API service to pass new filter parameters.

**No migration required** — No new database columns or tables needed; changes are to query logic, locking strategy, and UI display.
**No webhook contract changes** — MuleSoft endpoints unaffected.
**No breaking API changes** — Only additive fields in response DTOs.
**No simulation logic changes** — Seat allocation logic is preserved; only the concurrency guard is added.
