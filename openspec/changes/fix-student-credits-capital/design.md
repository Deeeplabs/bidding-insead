## Context

The student dashboard and bidding round views display capital and credits information. After the initial implementation of campaign-based accumulation logic, **three code paths still allow negative or incorrect values**:

1. `StudentCapitalService::getCapital()` — computes `capital - spent` without floor
2. `CampaignToActiveBiddingRoundDtoMapper` — computes `capitalGranted - cumulativePoints` without floor
3. `StudentCreditService::getCredits()` — returns hardcoded `left: 1` instead of actual calculation

The frontend expects existing fields (capital_spent, capital_left, credits_earned, credits_to_be_fulfilled) — only the backend calculation logic needs to be fixed.

## Goals / Non-Goals

**Goals:**
- Ensure `capital_left` is never negative across ALL code paths (dashboard stats, student detail, student list, bidding round module)
- Fix the hardcoded `left: 1` credits placeholder in `StudentCreditService`
- Maintain backward compatibility with existing API response shape
- Keep existing `max(0, ...)` guards where already present (`StudentStatsService`, `StudentCreditService::getCredits()` for `toBeFulfilled`)

**Non-Goals:**
- No changes to frontend (bidding-web)
- No changes to DTO field names or types
- No new API endpoints
- No database schema changes

## Decisions

### 1. Capital Floor Strategy — Apply at Model Level
**Decision**: Add `max(0, ...)` guard in `StudentCapitalService::getCapital()` at the `StudentCapital` model creation point (line 44).

**Rationale**: This is the single source of truth for capital calculations consumed by `StudentStatsService`, `StudentToDtoMapper`, and `StudentListToDtoMapper`. Fixing here prevents negative capital across all three consumers without modifying each individually.

### 2. Bidding Round Capital Floor — Apply at Mapper Level
**Decision**: Add `max(0, ...)` guard in `CampaignToActiveBiddingRoundDtoMapper::buildBiddingRoundModuleData()` (line 402).

**Rationale**: This is a separate code path that calculates per-module capital_left using campaign-level `min_capital_per_student` config, not the accumulated capital from `StudentCapitalService`. Must be fixed independently.

### 3. Credits Left — Proper Calculation
**Decision**: Replace hardcoded `left: 1` in `StudentCreditService::getCredits()` (line 60) with a proper calculation based on `totalCredits - totalCreditsEarned`, clamped to `max(0, ...)`.

**Rationale**: The `left` field in `StudentCredit` model should represent remaining credits available. The hardcoded `1` appears to be a placeholder from initial development.

## Risks / Trade-offs

**[Risk] Edge case: Student overspending capital** → **Mitigation**: `max(0, ...)` ensures no negative display. The system already prevents overspending via bid validators (`BidPointValidator`), so this is a display-only safeguard.

**[Risk] Regression in other capital consumers** → **Mitigation**: The fix in `StudentCapitalService` is at the model layer, so all downstream consumers (stats, detail, list) automatically get the fix. No individual consumer changes needed.

**[Risk] Students with no bids** → **Mitigation**: Returns 0 for capital_left and credits_to_be_fulfilled when no campaigns found (existing behavior, unchanged).

**[Trade-off] Credits `left` field semantics** → The `left` field on `StudentCredit` could mean either "credits remaining to take" or "credit slots remaining". We're implementing it as `max(0, totalCredits - creditsEarned)`, which means "credits remaining to take". This aligns with how `toBeFulfilled` already works.
