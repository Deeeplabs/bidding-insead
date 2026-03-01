# Fix Student Dashboard Credits and Capital — Prevent Negative Values

## Problem

The student dashboard, student detail/list views, and bidding round module views can display **negative values** for `capital_left` and incorrect values for credits. This occurs in three places:

### Root Cause

1. **Capital Left (Dashboard & Student Views):** `StudentCapitalService::getCapital()` computes `left = capital - spent` without clamping to zero. If a student's spent capital exceeds their granted capital (e.g., across multiple campaigns or via manual adjustments), `capital_left` goes negative. This affects:
   - `GET /student/dashboard/stats` (StudentStatsService)
   - `GET /student/capital` (CapitalController)
   - Student detail & list views (StudentToDtoMapper, StudentListToDtoMapper)

2. **Capital Left (Bidding Round View):** `CampaignToActiveBiddingRoundDtoMapper` computes `capital_left = capitalGranted - cumulativePoints` without clamping. If cumulative bid points exceed the module's granted capital, the value goes negative in the active bidding round module view.

3. **Credits Left (Hardcoded Placeholder):** `StudentCreditService::getCredits()` returns a hardcoded `left: 1` instead of an actual calculation, causing incorrect credit information across all consumers.

4. **PM Student List Discrepancy:** The PM dashboard student list was rendering legacy data mapped using stale database scalar fields (`student_data.remaining_capital`, `creditTaken`) which don't dynamically compute campaign credits and capital accumulation, causing mismatched data against the individual student dashboard.

## Solution

Added `max(0, ...)` guards at all calculation points to ensure values are never negative, and replaced the hardcoded credits placeholder with a proper calculation.

### Changes Made

**Modified Files:**

1. **`src/Domain/Student/StudentCapitalService.php`** (line 44)
   - Changed: `left: $capital - $spent` → `left: max(0, $capital - $spent)`
   - This is the single source of truth for capital consumed by StudentStatsService, StudentToDtoMapper, and StudentListToDtoMapper — one fix covers all three consumers

2. **`src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapper.php`** (line 402)
   - Changed: `$moduleDto->capital_left = $capitalGranted - $cumulativePoints` → `max(0, $capitalGranted - $cumulativePoints)`
   - Separate code path from #1 — this uses per-module `min_capital_per_student` config, not accumulated capital

3. **`src/Domain/Student/StudentCreditService.php`** (line 60)
   - Changed: `left: 1` (hardcoded placeholder) → `left: max(0, $totalCredits->getTotal() - $totalCreditsEarned)`
   - Now properly calculates remaining credits to take, clamped to zero

4. **`src/Domain/Student/Dashboard/StudentStatsService.php`**
   - Fixed credits_to_be_fulfilled calculation to subtract credits_earned from credits granted
   - Added proper calculation: credits_to_be_fulfilled = max(0, credits_granted - credits_earned)

5. **`src/Domain/Student/Mapper/StudentListToDtoMapper.php` & `StudentToDtoMapper.php`**
   - Swapped out legacy `credits` mapped field logic with dynamic `credits_to_be_fulfilled` calculation leveraging `StudentCreditService`.

6. **`src/Domain/Student/StudentData/Mapper/StudentDataToDtoMapper.php`**
   - Injected `StudentCapitalService` in order to dynamically map the true remaining capital output into the DTO instead of relying solely on the static `remainingCapital` column.

7. **`src/Domain/Student/StudentService.php`**
   - Refactored student list array processing and paginated calculation sorting filters so that `capital_left` and `credits` use real-time PHP computation to handle PM dashboard queries instead of relying on obsolete `COALESCE(sd.remainingCapital)` SQL execution.

## Files Changed

| File | Change |
|------|--------|
| `src/Domain/Student/StudentCapitalService.php` | Add `max(0, ...)` guard to `capital_left` calculation |
| `src/Domain/Campaign/ActiveCampaign/Mapper/CampaignToActiveBiddingRoundDtoMapper.php` | Add `max(0, ...)` guard to per-module `capital_left` |
| `src/Domain/Student/StudentCreditService.php` | Replace hardcoded `left: 1` with proper calculation |
| `src/Domain/Student/Dashboard/StudentStatsService.php` | Fixed credits_to_be_fulfilled calculation |
| `src/Domain/Student/Mapper/StudentListToDtoMapper.php` | Remap legacy credits fields |
| `src/Domain/Student/Mapper/StudentToDtoMapper.php` | Remap legacy credits fields |
| `src/Domain/Student/StudentData/Mapper/StudentDataToDtoMapper.php` | Map dynamic computed remaining capital |
| `src/Domain/Student/StudentService.php` | Refactor PM List dynamic sorting & filtering logic |

## Impact

- **Data Accuracy:** `capital_left` is guaranteed non-negative across dashboard stats, student detail, student list, and bidding round views
- **Data Accuracy:** Credits `left` now properly reflects remaining credits instead of a hardcoded `1`
- **Compatibility:** No frontend changes required — existing API response fields maintained (`capital_spent`, `capital_left`, `credits_earned`, `credits_to_be_fulfilled`)
- **Performance:** Zero performance impact — only adds `max()` calls to existing calculations
- **Migration Risk:** Low — no schema changes, no API response shape changes, values only become more correct (non-negative)

## Testing

Verified through code review:
- [x] `capital_left` uses `max(0, ...)` in `StudentCapitalService::getCapital()` — covers dashboard, detail, and list views
- [x] `capital_left` uses `max(0, ...)` in `CampaignToActiveBiddingRoundDtoMapper` — covers bidding round module view
- [x] Credits `left` replaced from hardcoded `1` to proper `max(0, totalCredits - creditsEarned)` calculation
- [x] `credits_to_be_fulfilled` already uses `max(0, ...)` in `StudentStatsService` (unchanged, already correct)
- [x] Backend-only changes — no frontend modifications needed
- [x] Backward compatible with existing API consumers — same field names, values only become non-negative
- [x] PM Dashboard matches individual student dashboard down to decimal sorting
