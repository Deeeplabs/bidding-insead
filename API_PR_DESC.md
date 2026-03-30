# Fix: Bid Points Capital Validation and Dummy Data Generation
 
 ## Problem
 Jira: https://insead.atlassian.net/browse/DPBFAD-811
 
 QA reported that "Generate Bid Data" produces bids with values less than 200 (the configured min_capital_per_student). Analysis revealed several underlying flaws causing this and other logic failures in `DummyDataService.php` and `BidValidator.php`:
 
 1. **Incorrect Config Fallback**: Capital validations were falling back to `min_bids_entire_round` if `min_capital_per_student` was not present or zero. This incorrectly forced a capital minimum to match a section count minimum (e.g., 5).
 2. **Confusing Shortfall Distribution**: DummyDataService enforced the minimum capital by dumping the entire shortfall onto the very last bid generated, leaving the other bids at very low random numbers (e.g., 2 or 4). QA looking at PM dashboards for specific courses saw these tiny bids and assumed the 200 capital constraint was violated entirely.
 3. **Draft Mutation Bug**: The dummy generator mutated the outer loop's `$bidStatus` variable. If one student's generated courses didn't meet the minimum credits limit, *every subsequent student* was permanently set to generate draft bids.
 4. **Duplicate Classes**: The `ClassPromotions` query fetched multiple sections for the same course, allowing a single student to bid on the same course multiple times.
 
 ## Goal
 Ensure dummy generated data strictly adheres to constraints, removes visual anomalies confusing QA, fixes critical iteration bugs, and removes the illogical fallback in the main API submission validator.
 
 ## Changes Made
 
 1. **`src/Domain/Campaign/ActiveCampaign/Validator/BidValidator.php`**
    - Fixed `validateCapital()`: removed the fallback to `min_bids_entire_round`. Capital bounds now rely strictly on capital configurations.
 
 2. **`src/Service/DummyDataService.php`**
    - Removed the fallback to `min_bids_entire_round` and `max_bids_entire_round` from effective capital calculation.
    - Implemented a locally scoped clone for the `$bidStatus` to prevent 'draft' overrides from leaking into subsequent loop iterations.
    - Added a `$selectedCourseIds` tracking array to filter out duplicate assigned sections belonging to the same course.
    - Updated the Minimum Capital Enforcer to spread out the required minimum points across *all allocated bids* relatively evenly (using random scaling up to ~50 points increments). This ensures that no individual course jumps erratically to 198 points while the rest sit at 2; the points are now believable and much more consistent, solving the visual test discrepancies for QA.
 
 ## Impact
 - Bid validation now correctly enforces the admin-configured minimum capital (`min_capital_per_student`) for real student bid submissions cleanly.
 - Generated dummy bid data will visually and technically respect min/max capital ranges.
 - Fixed dummy bid status generation leaks.
 - Resolved multiple-section duplicate assignment for dummy data.
 - No database migration, entity changes, or frontend changes required.
