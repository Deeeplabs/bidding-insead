## Context

The INSEAD bidding system displays seat information in two contexts: the PM's admin dashboard (bidding-admin) and the student portal (bidding-web). Currently, the student-facing `CourseOptionContent.tsx` shows a dynamic chip (`available_seats/total_seats seats`) that updates as students enroll — revealing remaining seat counts in real-time. The PM's dashboard (`course-table-bidding-round.tsx`) shows a "Seat Available" column that displays `XX (XX)` whenever `total_all_seats !== total_seats`, which is not the correct condition for the split format.

The updated requirement is:
1. **Students** see only a single static total-seats value during both Bidding Round and Add/Drop — never remaining seats or consumption.
2. **PM dashboard** shows `XX` for non-shared courses and `XX (XX)` only when a course is genuinely shared/split across multiple programme-promotions.
3. **Backend** continues to enforce real-time seat validation to prevent over-enrollment, returning a clear error when a course is full.

**Affected apps**: bidding-web (student seat display), bidding-admin (PM seat column format), bidding-api (ensure proper data for split detection + enrollment validation).

## Goals / Non-Goals

**Goals:**
- Replace dynamic seat chip in student UI with a static total-seats-only display
- Fix PM dashboard seat column to show `XX (XX)` only for courses shared across multiple programme-promotions
- Ensure backend enrollment validation prevents over-enrollment with clear error messaging
- Apply to both Bidding Round and Add/Drop phases consistently

**Non-Goals:**
- Adding new seat-related API fields (existing fields are sufficient)
- Adding sort/filter by availability (students should not see availability)
- Changing the simulation engine, waitlist logic, or capital allocation rules
- Modifying webhook contracts or MuleSoft integrations
- Real-time push/WebSocket updates (not needed since display is static)

## Decisions

### Decision 1: Static seat display in student UI — show only `total_seats`

**Choice**: In `CourseOptionContent.tsx`, replace the dynamic `{data.available_seats}/{data.total_seats} seats` chip with a static `{data.total_seats} seats` chip. Remove conditional coloring based on `available_seats > 0`. The chip always shows the configured total capacity in a neutral style.

**Why not remove the chip entirely?** Students still need to know how many seats the course offers to make informed decisions. The requirement states students should see the "total number of seats offered" — just not remaining/consumed counts.

**Alternative considered**: Removing `available_seats` and `enrolled_count` from the API response entirely. Rejected because this would be a breaking API change and other consumers (admin views, reports) may use these fields. Instead, we only change what the student UI renders.

### Decision 2: PM dashboard split format based on class promotion count

**Choice**: In `course-table-bidding-round.tsx`, the current condition `row.total_all_seats && row.total_all_seats !== row.total_seats` is incorrect — it shows the split format whenever the values differ, even for non-shared courses. Replace this with a check for whether the course is associated with multiple programme-promotions.

**Implementation approach**: The API already provides `total_seats` (adjusted/promotion-specific) and `total_all_seats` (sum across all promotions from `ClassPromotions`). A course is "shared/split" when it has `ClassPromotions` records across more than one programme-promotion. Add an `is_shared` boolean to `AdminBiddingRoundCourseDto` computed from `count(ClassPromotions) > 1` or `count(distinct promotions) > 1`. The admin frontend uses `is_shared` to decide the format:
- `is_shared === false` → display `total_seats`
- `is_shared === true` → display `total_seats (total_all_seats)`

**Why a dedicated boolean over reusing existing fields?** The current heuristic (`total_all_seats !== total_seats`) is unreliable. A course may have a single promotion but an adjusted seat capacity that differs from the original. An explicit `is_shared` flag is unambiguous.

### Decision 3: Preserve backend enrollment validation as-is

**Choice**: The existing `AddDropService.executeCampaignAddDrop()` already has pessimistic locking (`SELECT ... FOR UPDATE`) and seat capacity checks. No changes needed to the validation logic itself. Only verify that when seats are exhausted, the error message returned to the student is clear (e.g., "Course is full" or equivalent).

**Why no changes?** The backend already correctly prevents over-enrollment. The previous implementation tasks (2.1, 2.2) added pessimistic locking. The concern now is purely about what the UI displays — the backend is sound.

### Decision 4: Scope includes both Bidding Round and Add/Drop

**Choice**: Apply static seat display to both phases. In `CourseOptionContent.tsx`, the seat chip is used in both the bidding round course selector and the add/drop course selector (via `CourseSelector.tsx` → `ClassSelector.tsx`). A single change to `CourseOptionContent.tsx` covers both phases.

**Why not phase-specific logic?** The same component (`CourseOptionContent`) renders course options in both contexts. Applying different display logic per phase would add unnecessary complexity and diverge from the PM's requirement of consistent static display across all phases.

## Risks / Trade-offs

**[Risk] Students may not realize a course is full until they try to enroll** → This is by design per PM requirements. The system returns a clear error ("Course is full") at enrollment time. The UX trade-off is intentional: simplicity over real-time feedback.

**[Trade-off] `is_shared` flag adds a new field to admin API response** → This is an additive change (no existing fields removed). The field is only used by the admin dashboard. If the admin frontend is not deployed simultaneously, the old behavior (showing `XX (XX)` based on value difference) continues until the frontend update is deployed.

**[Trade-off] API still returns `available_seats` and `enrolled_count`** → These fields remain in the API response for backward compatibility and potential use by other consumers. Only the student UI stops rendering them. This avoids a breaking API change while achieving the desired UX.

**[Risk] PM dashboard column label says "Seat Available" but now shows static configured capacity** → The column label may need updating to "Seats" or "Seat Capacity" to avoid confusion. This is a minor cosmetic change in `course-table-bidding-round.tsx`.
