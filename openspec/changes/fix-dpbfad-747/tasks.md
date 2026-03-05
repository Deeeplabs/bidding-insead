## 1. Fix useEffect Query Payload Rebuild

- [x] 1.1 In `student-list-table.tsx`, replace the `setQueryPayload` call inside the large `useEffect` (lines 373-387) to build a clean payload object without `...prev` spread. Set `page: 1` instead of `page: prev?.page || 1`. Explicitly preserve `search` from `prev` if present. Remove stale filter keys by not spreading prev — instead construct the object with only the keys that are actively set.

  **File:** `bidding-admin/src/components/campaign/student/student-list-table.tsx`

- [x] 1.2 Add `currentSort` to the `useEffect` dependency array (line 389-397) to fix the stale closure for `sort` and `order` params.

  **File:** `bidding-admin/src/components/campaign/student/student-list-table.tsx`


## 2. Fix handleClear Function

- [x] 2.1 Refactor `handleClear` (lines 269-275) to use functional state update pattern `setQueryPayload((prev) => ...)` instead of stale closure `...queryPayload`. Reset `page` to 1. Only preserve `promotion_id`, `sort`, and `order` from prev — clear all filter keys.

  **File:** `bidding-admin/src/components/campaign/student/student-list-table.tsx`


## 3. Verification

- [ ] 3.1 Manual verification: Navigate to Create Campaign page, load student list, go to page 2 or 3, then apply a filter (e.g., campus filter). Verify the page resets to 1 and correct filtered data is displayed.

- [ ] 3.2 Verify filter removal: Apply a campus filter, then remove it. Verify `campus_ids` is no longer sent in the API request (check Network tab) and pagination totals reflect unfiltered data.

- [ ] 3.3 Verify exclude/include: Exclude a student while on page 2+. Verify page resets to 1 and the excluded student is no longer shown.

- [ ] 3.4 Verify sort persistence: Sort by a column, then apply a filter. Verify the sort order is maintained after the filter is applied.

- [ ] 3.5 Verify clear: Click clear after applying filters while on page 3. Verify page resets to 1 and all filters are removed.

- [ ] 3.6 Verify basic pagination: Without any filters, click through pages 1, 2, 3. Verify correct data loads for each page.
