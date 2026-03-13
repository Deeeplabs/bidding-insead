## Context
Currently, the student dashboard header may not accurately reflect the allocated credits and capital, or what has been spent/fulfilled during bidding cycles. The values are not correctly taking the rules set by the Program Manager (PM) for minimum credits and maximum capital. Credits earned must include final enrollment. 

## Goals / Non-Goals

**Goals:**
- Update `StudentCreditService.php` and `StudentCapitalService.php` domain calculations.
- Ensure "Credit Earned" (`credits_earned` / `credits`) dynamically computes adjusting for manual adjustments and final enrollment course credits across 2 bidding rounds.
- Ensure "Credits to be fulfilled" dynamically extracts the minimum credits requested by the PM campaign configuration for the active bidding cycles.
- Extend `StudentToDtoMapper.php` and `StudentListToDtoMapper.php` logic to accurately feed `capital_left`, `capital_spent`, and `credits` directly to the PM dashboard (Admin Frontend) from these updated domain methods. 
- Ensure "Capital Spent" strictly evaluates the bid points spent in the 2 bidding rounds.

**Non-Goals:**
- No changes to UI layer representation in `bidding-web` or `bidding-admin` assuming API payload structure remains unchanged.
- No DB migrations.

## Decisions
- Update central functions in `StudentCreditService` and `StudentCapitalService` rather than separate Dashboard services to maintain parity between Student Dashboard and PM Dashboard.
- Modify `StudentDto` and mapper configurations if "Credits to be fulfilled" is needed separately on the PM end, or ensure existing API attributes accurately echo PM configs.

## Risks / Trade-offs
- Risk: Changing existing calculation logic might temporarily affect in-flight dashboards. Mitigation: Ensure that PM configuration data is always correctly fetched and used.
- Risk: The API payload should remain compatible with `bidding-web`. Mitigation: Retain the existing fields in the response.
