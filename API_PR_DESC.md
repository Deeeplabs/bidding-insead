# Fix: Student Timezone Display - Backend Implementation

## Problem
Jira: [DPBFAD-919](https://insead.atlassian.net/browse/DPBFAD-919)

The Student Portal was displaying incorrect times (8-hour shift and DST-related shifts) for bidding rounds due to "Fake UTC" strings originating from the backend. The API was appending a `Z` suffix to dates formatted in the server's local timezone without prior conversion to UTC.

### 3. Capital (Bid Points) Check Disabled
`AddDropValidator::validateBidPoints()` had correct logic but was never called in `AddDropService::submitAddDrop()`. This loophole allowed submissions resulting in negative bid point balances.

### 1. `bidding-api/src/Helper/DateHelper.php`
- Refactored `toIso()` to perform an explicit conversion to UTC before formatting with the `Z` suffix.
- Handles both `\DateTime` and `\DateTimeImmutable` objects correctly.

### 2. `bidding-api/src/Controller/Api/Student/Campaign/StudentActiveCampaignController.php`
- Replaced manual `.format('Y-m-d H:i:s')` calls for campaign deadlines with `DateHelper::toIso()`.
- Standardized the API output for all student portal endpoints to ensure true ISO-8601 UTC strings are sent to the frontend.

## Impact
- **No Migrations**: This change affects only the presentation layer of the API.
- **Improved Data Integrity**: The API now provides a consistent, true UTC "source of truth", which is the standard requirement for cross-regional campus support.
- **Frontend Sync**: Directly enables the frontend display fix by removing the source of incorrect timezone offsets.

## Verification Steps
1. Perform a `GET` request to `/student/active-campaigns/{campaignId}`.
2. Verify that `start_date`, `end_date`, and `course_class_deadline` strings now end in `.000Z` and reflect the correct UTC time (not local server time).
3. Confirm that students in SGT now see the intended times without the 8-hour shift.
