## Context

The student dashboard displays capital and credits information. The current calculations may not accurately reflect the accumulation from campaigns the student has participated in. The frontend expects existing fields (capital_spent, capital_left, credits_earned, credits_to_be_fulfilled) - only the backend calculation logic needs to be fixed.

## Goals / Non-Goals

**Goals:**
- Fix capital_left calculation to use accumulation from campaigns participated + manual adjustments - spent
- Fix credits_to_be_fulfilled calculation to use minimum credits from campaigns participated - credits earned
- Use Bid entity to find campaigns the student has participated in
- Maintain backward compatibility with existing API response

**Non-Goals:**
- No changes to frontend (bidding-web)
- No changes to DTO fields
- No new API endpoints

## Decisions

### 1. Capital Calculation Strategy
**Decision**: Calculate capital as cumulative minCapitalGranted from all campaigns the student has placed bids in, plus manual adjustments, minus spent capital.

**Rationale**: This follows the accumulation model - students accumulate capital across campaigns they participate in.

### 2. Credits Calculation Strategy
**Decision**: Calculate credits_to_be_fulfilled as cumulative minCreditsToFulfill from campaigns the student has participated in, minus credits earned.

**Rationale**: Students must fulfill minimum credits from each campaign they participate in.

### 3. Finding Participated Campaigns
**Decision**: Use BidRepository to find distinct campaigns where student has placed bids.

**Rationale**: Direct and efficient way to find campaigns the student has actually participated in.

## Risks / Trade-offs

**[Risk] Query Performance** → **Mitigation**: Add database indexes on student_id, campaign_id if needed.

**[Risk] Students with no bids** → **Mitigation**: Return 0 for capital_left and credits_to_be_fulfilled when no campaigns found.

## Migration Plan

1. Modify StudentCapitalService to use BidRepository
2. Modify StudentCreditService to use BidRepository
3. Test with various scenarios
4. Deploy to DEV/INT/UAT/PROD

## Open Questions

None - implementation is straightforward.
