## Context

The `GET /v2/api/campaigns` endpoint returns per-campaign statistics via `CampaignService::calculateCampaignStatistics()`.

### The 3 Scenarios

| # | State | Expected Stats |
|---|-------|---------------|
| **Tier 1** | Pre-bidding OR bidding round is currently active (`start_date <= now <= end_date`) | Aggregate eligible students + completed across ALL active pre_bidding + bidding_round phases |
| **Tier 2** | Pre-bidding/bidding completed, but a downstream module (simulation, add_drop_waitlist) is still active | Calculate stats from the **closest past bidding round** (fallback) |
| **Tier 3** | No phase of any type is currently active (all completed or all future) | Return `null` for all three stats (frontend shows "-") |

### Why Tier 2 Matters

When bidding round completes, the campaign moves to simulation → add/drop → final enrollment. During these phases, the bidding data is still relevant (students enrolled, bids completed). The stats should still show the bidding round's data because the campaign is still "in progress."

## Goals / Non-Goals

**Goals:**
- Implement 3-tier stats calculation: active bidding → past bidding fallback → null
- Aggregate across multiple active pre_bidding/bidding_round phases (Tier 1)
- Fall back to closest past bidding round when only non-bidding modules are active (Tier 2)
- Return null when no phase is truly active (Tier 3)
- Use direct phase-date iteration for active-phase detection (bypass `final_enrollment` fallback)

**Non-Goals:**
- Modifying Campaign entity methods
- Frontend changes

## Decisions

### 1. Three-tier calculation in a single method

**Decision:** Implement all 3 tiers in `calculateCampaignStatistics()`:

```php
private function calculateCampaignStatistics(Campaign $campaign): array
{
    $now = new \DateTime();
    $hasActivePhase = false;
    $activeBiddingPhases = [];

    // Single iteration: detect active phases AND collect pre_bidding/bidding_round phases
    foreach ($campaign->getExecutions() as $execution) {
        if ($execution->isDeleted()) continue;
        $campaignModule = $execution->getCampaignModule();
        if (!$campaignModule || $campaignModule->isDeleted()) continue;
        $moduleCode = $campaignModule->getModule()?->getCode();

        foreach ($execution->getPhaseConfigs() as $phaseConfig) {
            if ($phaseConfig->isDeleted() || !$phaseConfig->isActive()) continue;
            $startDate = $phaseConfig->getStartDate();
            $endDate = $phaseConfig->getEndDate();

            if ($startDate && $endDate && $now >= $startDate && $now <= $endDate) {
                $hasActivePhase = true;

                if (in_array($moduleCode, ['pre_bidding', 'bidding_round'])) {
                    $moduleConfig = $phaseConfig->getModuleConfig();
                    $activeBiddingPhases[] = [
                        'campaign_module_id' => $campaignModule->getId(),
                        'phase_id' => $phaseConfig->getId(),
                        'module_code' => $moduleCode,
                        'config' => array_merge(
                            $moduleConfig['config'] ?? [],
                            isset($moduleConfig['student_selection'])
                                ? ['student_selection' => $moduleConfig['student_selection']]
                                : []
                        ),
                    ];
                }
            }
        }
    }

    // TIER 3: No phase is truly active → return null
    if (!$hasActivePhase) {
        return [
            'total_students' => null,
            'bidding_pending_count' => null,
            'bidding_completed_count' => null,
        ];
    }

    // TIER 1: Active pre_bidding/bidding_round phases exist → aggregate
    if (!empty($activeBiddingPhases)) {
        return $this->aggregateBiddingStats($campaign, $activeBiddingPhases);
    }

    // TIER 2: Non-bidding module active (simulation/add_drop), bidding completed
    // → fall back to closest past bidding round
    $biddingActive = $campaign->getCurrentActivePhaseDetailsByModule('bidding_round', true);
    if ($biddingActive !== null) {
        return $this->calculateStatsFromPhase($campaign, $biddingActive);
    }

    // Fallback: active non-bidding module but no past bidding round found
    return [
        'total_students' => 0,
        'bidding_pending_count' => 0,
        'bidding_completed_count' => 0,
    ];
}
```

**Rationale:** Clean separation of the 3 tiers, with the iteration loop handling both null detection and phase collection in one pass.

### 2. Helper methods for stats calculation

**Decision:** Extract two helper methods to keep the main method clean:

1. `aggregateBiddingStats($campaign, $activeBiddingPhases)` — Tier 1: loop through all active phases, sum eligible students + completed bids
2. `calculateStatsFromPhase($campaign, $phaseDetails)` — Tier 2: calculate stats from a single phase (the past bidding round)

```php
private function aggregateBiddingStats(Campaign $campaign, array $activeBiddingPhases): array
{
    $totalStudents = 0;
    $biddingCompletedCount = 0;

    foreach ($activeBiddingPhases as $phase) {
        $eligibleStudents = $this->campaignStudentEligibilityService->getEligibleStudents(
            $campaign, $phase['config'], $phase['phase_id']
        );
        $totalStudents += is_array($eligibleStudents) ? count($eligibleStudents) : 0;

        if ($phase['module_code'] === 'pre_bidding') {
            $completedSubmissions = $this->courseRankingRepository
                ->findCompletedSubmissionByCampaign($campaign->getId()) ?? [];
            $phaseCompleted = count($completedSubmissions);
            if ($phaseCompleted === 0) {
                $completedBids = $this->bidRepository->findTotalBidCompletedByCampaign(
                    $campaign->getId(), (int) $phase['campaign_module_id'],
                    (int) $phase['phase_id'], 'pre_bidding'
                ) ?? [];
                $phaseCompleted = count($completedBids);
            }
            $biddingCompletedCount += $phaseCompleted;
        } else {
            $completedBids = $this->bidRepository->findTotalBidCompletedByCampaign(
                $campaign->getId(), (int) $phase['campaign_module_id'],
                (int) $phase['phase_id'], $phase['module_code']
            ) ?? [];
            $biddingCompletedCount += count($completedBids);
        }
    }

    $biddingPendingCount = max(0, $totalStudents - $biddingCompletedCount);
    return [
        'total_students' => $totalStudents,
        'bidding_pending_count' => $biddingPendingCount,
        'bidding_completed_count' => $biddingCompletedCount,
    ];
}

private function calculateStatsFromPhase(Campaign $campaign, array $phaseDetails): array
{
    $totalStudents = 0;
    $config = $phaseDetails['config'] ?? null;
    $phaseConfigId = $phaseDetails['phase_id'] ?? null;

    if ($config !== null && $phaseConfigId !== null) {
        $eligibleStudents = $this->campaignStudentEligibilityService->getEligibleStudents(
            $campaign, $config, $phaseConfigId
        );
        $totalStudents = is_array($eligibleStudents) ? count($eligibleStudents) : 0;
    }

    $biddingCompletedCount = 0;
    $biddingModuleId = $phaseDetails['campaign_module_id'] ?? null;
    $biddingPhaseId = $phaseDetails['phase_id'] ?? null;
    $biddingModuleCode = $phaseDetails['module_code'] ?? 'bidding_round';

    if ($biddingModuleId && $biddingPhaseId) {
        $completedBids = $this->bidRepository->findTotalBidCompletedByCampaign(
            $campaign->getId(), $biddingModuleId, $biddingPhaseId, $biddingModuleCode
        ) ?? [];
        $biddingCompletedCount = count($completedBids);
    }

    $biddingPendingCount = max(0, $totalStudents - $biddingCompletedCount);
    return [
        'total_students' => $totalStudents,
        'bidding_pending_count' => $biddingPendingCount,
        'bidding_completed_count' => $biddingCompletedCount,
    ];
}
```

### 3. Using getCurrentActivePhaseDetailsByModule() for Tier 2 only

**Decision:** The entity method `getCurrentActivePhaseDetailsByModule('bidding_round', true)` (with `isClosestPast=true`) is used ONLY in Tier 2 — to find the closest past bidding round when no pre_bidding/bidding_round is currently active.

This is safe because:
- It's NOT used for null detection (Tier 3 uses direct phase-date iteration)
- It's NOT used for aggregation (Tier 1 uses collected phases)
- It specifically targets `bidding_round` module code — no `final_enrollment` fallback risk

### 4. File changes

| File | Change |
|------|--------|
| `CampaignService.php` | Rewrite `calculateCampaignStatistics()` with 3-tier logic, add `aggregateBiddingStats()` and `calculateStatsFromPhase()` helpers |
| `CampaignListDto.php` | Already nullable — no changes needed |

## Risks / Trade-offs

### Risk: `findCompletedSubmissionByCampaign()` is campaign-wide
- Returns ALL completed submissions for the campaign, not per-module
- When aggregating across multiple pre_bidding modules, may double-count
- **Mitigation:** In practice, campaigns typically have at most one pre_bidding module

### Risk: `getCurrentActivePhaseDetailsByModule('bidding_round', true)` returns stale data
- This is intentional for Tier 2 — we WANT the past bidding round data when a downstream module is active
- The `isClosestPast=true` parameter finds the most recently started bidding round
