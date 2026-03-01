# Fix Student Dashboard Credits and Capital — PM Rule Alignment

## Problem

The student dashboard, student detail/list views, and bidding round module views had incorrect definitions for certain campaign statistics and could display negative values for `capital_left`. Based on PM rules for scenarios such as ST 1 (1 campaign, 2 bidding rounds), the four main values must be calculated as follows:

1. **Credit Earned:** Must strictly sum the Student Final Enrollment Courses Credits for all involved rounds (e.g., 2 bidding rounds). The previous calculation incorrectly counted all bids during active periods instead of purely the finalized enrollment courses, resulting in inflated earned credits.
2. **Credits to be fulfilled:** Must sum the Minimum credits required per student for the entire bidding cycle (all rounds) when configured by PM. Previously, this was misconfigured as a 'Remaining' attribute (`Target - Earned`). It should be outputted strictly as the configured static target for the overall campaign cycle.
3. **Capital Spent:** Must sum the bid points spent across all rounds in the campaign (e.g., in 2 bidding rounds).
4. **Capital Left:** Must output the Total Capital granted to the student for the entire campaign when configured by PM minus the Capital Spent. Previously, this was computed without a floor (`capital - spent`). When points mathematically exceeded capital (e.g. manual adjustments), it went negative on dashboards and active module screens.

*Note on PM Student List Discrepancy:* The PM dashboard student list was pulling from legacy statically-mapped database states instead of calculating `Credits` and `Capital Left` dynamically to match these backend calculations.

## Solution

Aligned definitions strictly with PM requirements outlined in the ST 1 scenario (1 campaign, multiple rounds) and implemented a `max(0, ...)` floor for all math dealing with `Capital Left`.

### Changes Made

**Modified Files:**

1. **`src/Domain/Student/StudentCapitalService.php`**
   - Added `max(0, $capital - $spent)` to the root Model creation preventing negative capital anywhere.
   - Calculates **Capital Left** and **Capital Spent** dynamically over all campaign rounds.

2. **`src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapper.php`**
   - Added `max(0, $capitalGranted - $cumulativePoints)` floor for active bidding modules.

3. **`src/Domain/Student/StudentCreditService.php`**
   - Replaced hardcoded `left: 1` placeholder.
   - Fixed **Credit Earned** logic to rely explicitly on `getStudentTotalCreditsTaken($student)` so only finalized enrolled courses constitute "Earned" credits.

4. **`src/Domain/Student/Dashboard/StudentStatsService.php`**
   - Corrected **Credits to be fulfilled** logic to push the statically configured Campaign Target (`$creditsGranted`) directly to the frontend rather than an ever-decreasing remainder.

5. **`src/Domain/Student/Mapper/StudentListToDtoMapper.php` & `StudentToDtoMapper.php`**
   - Mapped `Credits` columns on the PM Dashboard to natively pipe through the true campaign target (`$creditsGranted`) exactly matching the Student UI expectations.

6. **`src/Domain/Student/StudentData/Mapper/StudentDataToDtoMapper.php`**
   - Injected `StudentCapitalService` in order to dynamically map true remaining capital into the DTO instead of relying solely on static dataset mapping.

7. **`src/Domain/Student/StudentService.php`**
   - Refactored sorting to correctly handle compute-based `$capital_left`.

## Impact

- **Data Accuracy:** `Credit Earned` strict constraints correctly calculate metrics across multiple bidding rounds.
- **Data Accuracy:** `Credits to be fulfilled` accurately surfaces as the PM-defined target.
- **Data Accuracy:** `Capital Spent` aggregates flawlessly across active bidding rounds.
- **Data Accuracy:** `capital_left` correctly executes granted minus spent and is mathematically prevented from displaying negatives.
- **Compatibility:** Fully compliant with existing API shapes and structures (`capital_spent`, `capital_left`, `credits_earned`, `credits_to_be_fulfilled`).

## Testing

Verified through code review:
- [x] Scenario ST 1: Credit Earned correctly evaluates final enrolled courses across 2 bidding rounds.
- [x] Scenario ST 1: Credits to be fulfilled surfaces the static target (min credits configuration for the entire bidding cycle).
- [x] Scenario ST 1: Capital Spent tallies bid points safely over 2 bidding rounds.
- [x] Scenario ST 1: Capital Left applies total granted minus spent clamped to non-negative correctly.
- [x] Backend-only changes — no frontend modifications required.
- [x] Backward compatibility preserved.
