## Why

Currently, the student dashboard header may not accurately reflect the allocated credits and capital, or what has been spent/fulfilled during bidding cycles. This change ensures that Program Managers (PMs) and students have accurate, real-time insight into correct bidding activity calculations based on the PM's core and bidding round configurations.

## What Changes

- Update core calculations in `StudentCreditService` and `StudentCapitalService` to accurately compute credits and capital based on the active PM configuration instead of static data.
- **Credit Earned**: Ensure calculations encompass final enrollment credits and manual adjustment credits for the 2 bidding rounds.
- **Credits to be fulfilled**: Retrieve properly from the "Minimum credits required per student" PM campaign configuration for the entire bidding cycle.
- **Capital Spent**: Sum of bid points spent during the 2 bidding rounds.
- **Capital Left**: Total Capital granted to the student for the entire campaign (PM configured) minus Capital Spent.
- Both the **Student Dashboard API** and the **PM Dashboard API** (via `StudentDto`, `StudentListDto` mapping) will be updated to expose and reflect these exact definitions in real-time.

## Capabilities

### New Capabilities
- `student-dashboard-stats`: Accurately display credits (earned/fulfilled) and capital (granted/spent/left) based on PM calculations.

### Modified Capabilities

## Impact

- **bidding-api**: Updates to core Domain services (`StudentCreditService.php`, `StudentCapitalService.php`, `EnrollmentViewService.php`) and mapping DTOs (`StudentToDtoMapper.php`, `StudentListToDtoMapper.php`, `StudentStatsDto.php`).
- **bidding-admin**: Validates that PM view correctly displays updated capital and credits per student.
- **Data Integrity**: Enforces that PM configurations directly drive accurate financial and credit progress.
