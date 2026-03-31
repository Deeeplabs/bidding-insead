## Implementation: `bidding-web`

- [x] **Task 1: Refactor `CollapsePanelExtra.tsx`**
  - [x] Locate `src/features/bidding/components/shared/bidding-card/CollapsePanelExtra.tsx`.
  - [x] Modify lines 25-26 to use `parseUtc` directly for `parsedStartDate` and `parsedEndDate`.
  - [x] Avoid intermediate `format()` calls that lose timezone context.
  
- [x] **Task 2: Refactor `HeaderSection.tsx`**
  - [x] Locate `src/features/bidding/components/bid-submission/HeaderSection.tsx`.
  - [x] Modify lines 58-59 to use `parseUtc` directly for `parsedStartDate` and `parsedEndDate`.
  - [x] Verify that the `useMemo` dependencies for `phaseStatus` and `displayCountdown` are still correct.

## Post-Implementation Verification

- [x] **Task 3: Run existing logic tests**
  - [x] Run `npm unit test` on relevant utility tests in `bidding-web/src/features/bidding/utils/__tests__/phase.test.tsx`.
  - [x] Verify that `getPhaseStatus` and `getPhaseDisplayText` correctly handle both `Date` objects and ISO strings with 'Z'.

- [x] **Task 4: Manual Verification in Browser**
  - [x] PM (User profile set at SGT) create campaign with a round closing at 1:20 AM SGT.
  - [x] Login as Student (profile set at SGT).
  - [x] **Expected UI**: Bidding round card should show `Closes at 1:20 AM` or `Closed on 27 Mar 2026, 01:20 GMT+8`.
  - [x] **BUG UI**: Bidding round card should NOT show `09:20 GMT+8`.
