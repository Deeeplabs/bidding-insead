# bidding-duplicate-course-prevent

## Overview
Prevent users from submitting bids for the same exact course across multiple parallel bidding rounds (e.g., BIDDING1 and BIDDING2) during the Bidding phase.

## Requirements
1. **Detect Cross-Round Bidding Duplicates:** During the Active Bidding phase (`BidValidator` or `CreateBiddingService`), retrieve the student's existing bids in other concurrently active bidding campaigns within the same Program.
2. **Reject Duplicate Course Submissions:** If the student attempts to submit a bid for a course they have already bidded on in a parallel round, compilation should explicitly block it and throw an appropriate `DomainException` to avoid duplicates across parallel rounds.
3. **Validate Same-Submission Duplicates:** Additionally verify that the user cannot submit duplicate course IDs in the same single submission.
4. **Data Retrieval:** Add necessary database queries (`BidRepository`) to reliably fetch list of bidded course IDs across parallel rounds within the same overarching Program, bypassing the current active module.

## Example Flow
- User submits a bid for Course A in `BIDDING1`.
- User navigates to `BIDDING2` (within the same program) and tries to bid on Course A again.
- The system evaluates their pending bids across parallel rounds in `BidRepository` via `BidValidator`.
- The submission is rejected with an error message indicating the course has already been submitted in another round.

## Notes
- Needs regression tests for cross-round duplicate bidding submissions.