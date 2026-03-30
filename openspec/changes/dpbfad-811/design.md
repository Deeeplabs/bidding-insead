## Context

The QA raised DPBFAD-811 stating that "Generate Bid Data" produces bids with value less than 200 (the configured min_capital_per_student). Analysis revealed several underlying flaws in the `DummyDataService.php` and `BidValidator.php` logic:

1. **Incorrect Config Fallback**: `BidValidator` and `DummyDataService` were falling back to `min_bids_entire_round` (a count) if `min_capital_per_student` (a points sum) was not present or evaluated to 0. This incorrectly forced a capital minimum to match a section count minimum (e.g., 5).
2. **Confusing Shortfall Distribution**: DummyDataService tried to enforce the minimum capital, but dumped the entire shortfall onto the very last bid generated. This left the other bids at very low random numbers (e.g., 2 or 4). QA looking at PM dashboards for specific courses saw these tiny bids and assumed the 200 capital constraint was violated entirely.
3. **Draft Mutation Bug**: If a student's generated courses didn't meet the minimum credits limit, the dummy generator correctly set their bids to `'draft'`. But it mutated the outer loop's `$bidStatus` variable, meaning *every subsequent student* was permanently set to draft bids.
4. **Duplicate Classes**: The ClassPromotions query fetched multiple sections for the same course, and the dummy loop didn't de-duplicate by `CourseId`, allowing a student to bid on the same course multiple times.

## Goals / Non-Goals

**Goals:**
- Remove the illogical fallback to `min_bids_entire_round` for capital limits across the API.
- Fix DummyDataService `$bidStatus` scope mutation so student statuses are assessed independently.
- De-duplicate DummyDataService course assignments by `Course->getId()`.
- Distribute capital shortfall smoothly across all generated bids so the dummy data looks realistic to QA.

**Non-Goals:**
- Changing the admin UI or config DTO structure.
- Modifying the real bid submission endpoint contract.
- Changing entity or database schema.

## Decisions

**Decision 1: Eliminate fallback from capital to bids**
Remove `?? $config['min_bids_entire_round']` from capital resolution anywhere it appears. Capital (points) and Bids (count) are fundamentally different concepts and combining them under a fallback masks configuration omissions and lowers validation requirements improperly.

**Decision 2: Smooth Shortfall Distribution**
Instead of `$lastBid->bidPoints += $shortfall;`, implement a `while` loop that adds `mt_rand(1, min(50, $shortfall))` to each generated bid sequentially until the shortfall reaches 0. This creates realistic dummy bids (e.g., 40, 50, 60, 50) rather than standard deviation breaks (e.g., 2, 2, 2, 194).

**Decision 3: Deduplicate Course IDs**
Maintain a `$selectedCourseIds` array locally in the student iteration of `DummyDataService`, skipping any class whose `courseId` is already present. This fixes the multiple-sections-per-course bug in dummy generation.

## Risks / Trade-offs

- **[Risk] Stricter validation may reject bids that previously went through** -> This is intentional; falling back to a bid count when capital is intended effectively disabled capital bounds checks.
- **[Risk] Dummy generation takes slightly more CPU ops** -> Smoothed distribution requires an extra `while` loop over an array of size ~5-10, negligible performance impact.
