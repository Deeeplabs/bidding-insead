## Why

During both the Bidding Round and Add/Drop phases, seat availability is currently displayed dynamically to students — showing remaining seats (`available_seats/total_seats`) that change in real-time as other students enroll. This creates confusion, anxiety, and an inconsistent experience. The PM's dashboard also incorrectly shows the seat format: it currently displays `XX (XX)` for all courses with `total_all_seats`, but the `XX (XX)` format should only appear when a course is shared/split across multiple programme-promotions.

The PM requires that seat capacity be configured upfront and remain **static** in the UI. Students should only see the total number of seats offered — never remaining seats or consumption data. The system must still enforce real-time backend validation to prevent over-enrollment, but this should be invisible to the student until they attempt to enroll in a full course.

## What Changes

- **Make seat display static for students**: Replace the dynamic `available_seats/total_seats seats` chip in `CourseOptionContent.tsx` with a single static total-seats value. Students see only `XX seats` — never remaining or consumed seats.
- **Fix PM dashboard seat format**: The "Seat Available" column in the PM's bidding round course table (`course-table-bidding-round.tsx`) currently shows `XX (XX)` whenever `total_all_seats !== total_seats`. This should only display the split format `XX (XX)` when the course is genuinely shared across multiple programme-promotions — not merely when the values differ.
- **Enforce real-time backend enrollment validation**: Keep existing concurrency protection (`SELECT ... FOR UPDATE` in `AddDropService`) and seat capacity checks. When a course is full, return a clear error ("Course is full") and prevent enrollment. This validation is backend-only — the UI does not reflect real-time seat counts.
- **Remove dynamic seat fields from student-facing UI**: Stop displaying `available_seats` and `enrolled_count` in student-facing components. The API may still return these fields (backward compatibility), but the UI must not render them.

## Capabilities

### New Capabilities
- `static-seat-display`: Static seat capacity display for both Bidding Round and Add/Drop phases, with `XX` format for non-shared courses and `XX (XX)` format only for courses shared across programme-promotions.

### Modified Capabilities
_(No existing specs to modify — the openspec/specs/ directory is empty)_

## Impact

**bidding-api:**
- `Domain/Dashboard/Campaign/AdminCampaignDetailService.php` — Ensure `total_seats` and `total_all_seats` are set correctly so the PM dashboard can determine when to show split format. May need to add a flag or use class promotion count to determine if course is shared across programme-promotions.
- `Domain/Dashboard/Campaign/AdminBiddingRoundCourseDto.php` — May add an `is_shared_across_promotions` boolean field to explicitly signal when split format applies.
- `Domain/Campaign/ActiveCampaign/AddDropService.php` — Verify existing concurrency protection returns clear "Course is full" error message when seats are exhausted.

**bidding-admin:**
- `components/dashboard/process/bidding-round/course-table-bidding-round.tsx` — Fix the seat display condition: show `XX (XX)` only when the course is shared across multiple programme-promotions, not just when `total_all_seats !== total_seats`.

**bidding-web:**
- `features/bidding/components/bid-submission/CourseOptionContent.tsx` — Remove the dynamic `available_seats/total_seats seats` chip. Replace with a static `XX seats` chip showing only `total_seats`.
- `features/bidding/types/bidding.type.ts` — No type changes needed (fields remain for backward compatibility), but UI components must stop using `available_seats` and `enrolled_count`.
- `features/bidding/hooks/use-course-options.ts` — No changes needed; the `isFull` check via `STATUS_CLASS.FULL` already handles disabling full courses.

**No migration required** — No new database columns or tables needed; changes are to display logic only.
**No webhook contract changes** — MuleSoft endpoints unaffected.
**No breaking API changes** — Existing API response fields remain; only UI rendering changes.
**No simulation logic changes** — Backend enrollment validation is preserved as-is.
