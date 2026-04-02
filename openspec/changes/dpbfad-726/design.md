## Context

The INSEAD bidding system already supports direct enrollment during the Add/Drop phase via `AddDropService`. When a student adds a course and seats are available, the system assigns `ENROLLED` status immediately. However, several issues prevent this from working reliably:

1. **Silent course exclusion**: `AddDropAvailableCourseService` (lines 74-102) and `CampaignCourseService` (lines 48-256) apply filters (class status, promotion matching, ClassPromotions existence, non-zero seat checks) that silently exclude courses with available seats when configuration is incomplete or mismatched.
2. **No concurrency protection**: `AddDropService.executeCampaignAddDrop()` reads seat counts and creates bids without any database-level locking, allowing race conditions where two students both see 1 available seat and both enroll.
3. **Limited discoverability**: The student UI (`CourseSelector.tsx`) shows available courses in a dropdown but does not highlight seat availability or allow filtering/sorting by available seats.
4. **No real-time feedback**: After enrollment, the course list is not automatically refreshed — students may see stale seat counts until they navigate away and back.

**Affected apps**: bidding-api, bidding-web. bidding-admin is not directly affected.

## Goals / Non-Goals

**Goals:**
- Fix course filtering bugs so all courses with available seats appear in the add/drop course list
- Add pessimistic locking to prevent seat over-allocation during concurrent enrollments
- Surface seat availability (available/total) per section in the student add/drop UI
- Allow students to filter/sort courses by seat availability
- Invalidate React Query cache after enrollment to show updated seat counts immediately

**Non-Goals:**
- WebSocket/push-based real-time seat updates (polling on form interaction is sufficient)
- Changing the bidding phase behavior (only add/drop phase is in scope)
- Modifying the simulation engine or waitlist auto-promotion logic
- Admin-side UI changes
- Changing the `isFree` bid mechanics or capital allocation rules

## Decisions

### Decision 1: Fix filtering in AddDropAvailableCourseService with defensive fallbacks

**Choice**: Add fallback logic so that when `ClassPromotions` records are missing or promotion matching fails, the system logs a warning but still includes the course if seats are available (via `SimulationAdjustmentCourse` or `AdjustmentCourse` seat overrides).

**Why not just fix the data?** Data configuration issues will recur with each new campaign. A defensive backend that logs warnings but doesn't silently hide courses is more resilient. The root filtering logic should still prefer ClassPromotions-based seats when available, but fall back gracefully.

**Alternative considered**: Strict data validation at campaign creation time. Rejected because it doesn't fix courses already in-flight and adds admin friction.

### Decision 2: Pessimistic locking with SELECT ... FOR UPDATE on seat check

**Choice**: Wrap the seat availability check and bid creation in `AddDropService` within a single database transaction with `SELECT ... FOR UPDATE` on the relevant `Bid` rows for the class being enrolled. This prevents concurrent reads from seeing the same seat count.

**Why pessimistic over optimistic?** Add/drop operations are relatively low-frequency (not thousands per second), and pessimistic locking provides a stronger guarantee. Optimistic locking (version column) would require retry logic in the controller and a more complex error-handling path for the student.

**Implementation**: Use Doctrine's `LockMode::PESSIMISTIC_WRITE` in the repository method that counts enrolled bids for a class. The lock is held only for the duration of the enrollment transaction (milliseconds).

**Alternative considered**: Redis-based distributed lock. Rejected because MySQL transactions are sufficient for single-instance API and add no new infrastructure dependency.

### Decision 3: Add seat availability fields to existing available-course response DTO

**Choice**: Add `available_seats`, `total_seats`, and `enrolled_count` fields to the existing `AddDropAvailableCourseDto` response. These are additive fields — no existing fields are removed or renamed.

**Why not a separate endpoint?** The data is already computed in `AddDropAvailableCourseService` (lines 291-330). Adding it to the existing response avoids an extra API call and keeps the frontend simple.

### Decision 4: Add sort/filter query parameters to available-courses endpoint

**Choice**: Add optional `sort_by` (e.g., `available_seats_desc`, `course_name_asc`) and `availability` (e.g., `available`, `all`) query parameters to `AddDropAvailableCourseController`. Default behavior is unchanged (`all` courses, existing sort order).

**Why query parameters over client-side filtering?** The available courses endpoint is paginated (infinite scroll). Client-side filtering/sorting would only work on the loaded page, not the full dataset. Server-side ensures correct results across pages.

### Decision 5: React Query cache invalidation after enrollment mutation

**Choice**: After a successful add/drop submission, invalidate the `addDropAvailableCourses` query key so the course list refetches with updated seat counts. Also invalidate the student's enrollment list query.

**Why not optimistic updates?** Seat counts depend on server-side calculation (simulation adjustments, adjustment overrides). Optimistic updates could show incorrect numbers. A refetch after mutation is more reliable and the latency is acceptable (< 1 second).

## Risks / Trade-offs

**[Risk] Pessimistic lock contention under high load** → Lock is scoped to a single class's bid rows and held for < 100ms. At INSEAD scale (hundreds, not thousands of concurrent users), contention is negligible. If it becomes an issue, the lock scope can be narrowed further.

**[Risk] Defensive fallback may surface incorrectly configured courses** → The fallback adds a warning log. Admins should monitor logs and fix ClassPromotions configuration. The alternative (hiding courses) is worse for students.

**[Risk] Cache invalidation causes brief loading state in UI** → After enrollment submission, the course list will briefly show a loading indicator while refetching. This is acceptable UX and clearly communicates that data is being updated.

**[Trade-off] No real-time push updates** → Students must interact with the form (e.g., open course selector, scroll, search) to see updated seat counts. True real-time would require WebSocket infrastructure which is out of scope. The refetch-on-interaction approach is a pragmatic middle ground.

**[Trade-off] Existing filter behavior changes for all campaigns** → Fixing the promotion matching fallback affects all active campaigns, not just new ones. This is intentional — the current silent exclusion is a bug, not a feature. But we should test against production data to ensure no unexpected courses surface.
