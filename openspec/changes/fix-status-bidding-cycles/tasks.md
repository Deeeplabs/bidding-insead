## 1. Setup and Bug Reproduction

- [x] 1.1 Identify the controller/service invoked when the "Update Campaign" button is pressed for general properties (e.g., `CampaignUpdateService`), and pinpoint the logic that clears or recalculates phase status unintentionally.
- [ ] 1.2 Manually reproduce the bug in the local environment via the browser: create a campaign, start a Bidding phase, edit the main campaign minimum/max points, and confirm the Bidding phase incorrectly resets to "Not Started".

## 2. Core API Implementation

- [x] 2.1 Refactor the campaign update endpoint logic to protect the internal state array of `CampaignModule` and `CampaignPhase` objects from being overwritten or restored to defaults when unaffected payload fields are edited.
- [x] 2.2 Reevaluate and patch the progression sequence logic that opens next phases, addressing the issue where `Simulation` instantly skips to `Final Enrollment [CLOSED]` if the previous phase's status integrity was compromised.
- [x] 2.3 Ensure any internal database queries computing "Bidding Completed" and Add/Drop tables rely on existing stable records without treating a 0 count due to a phase status masking bug.

## 3. Testing and Verification

- [ ] 3.1 Perform manual browser verification from the video steps: start a Bidding phase, place bids (impersonating a student), edit the main campaign minimum/max points in the PM dashboard, and confirm the Bidding phase remains "Active" and bids are counted correctly.
- [ ] 3.2 Close the Bidding phase and check that the Simulation phase correctly opens and waits for user action, preventing jumps to Final Enrollment.
- [ ] 3.3 Verify Add/Drop phase statistics correctly reflect the bids and enrollments from previous phases without incorrectly displaying zero.
