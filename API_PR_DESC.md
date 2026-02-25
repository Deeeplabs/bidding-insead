# Fix Bidding Cycle Status Reset on Campaign Configuration Update

## Problem

When a Programme Manager updates the main campaign configuration (e.g., modifying "Minimum total points" or "Max credits" via the campaign Edit screen), any currently active phases (like a Bidding Round) are incorrectly reset to "Not Started".

**Impact:**
1. **State Overwrite & Data Masking**: Active phases lose their runtime state (`start_date`/`end_date`). Associated statistics (e.g., submitted student bids) display as 0, effectively hiding live data.
2. **Phase Progression Skipping**: After status corruption, progressing past the Bidding phase causes the Simulation phase to be skipped entirely — the campaign jumps directly to "Closed" Final Enrollment.

### Root Cause

The bug originates in two methods: `CampaignService::saveWithModules()` and `CampaignService::updateWithModules()`.

1. **Blind `module_config` overwrite on `CampaignModule`**: When the frontend submits a campaign update, it sends back all modules including their `module_config`. The backend was calling `$campaignModule->setModuleConfig($moduleData['module_config'])` which blindly replaced the config — wiping out runtime state like `start_date`/`end_date` set by `activatePhase()`.

2. **Unprotected phase date updates on `CampaignPhaseConfig`**: For existing phase configs, `start_date` and `end_date` from the request payload were applied unconditionally, overwriting dates set when the phase was activated/closed.

3. **Fragile Final Enrollment lookup**: The Final Enrollment status check used `$module->getId() - 1` (database auto-increment arithmetic) to find the previous module, which is unreliable when module IDs aren't sequential.

## Solution

### Changes Made

**File:** `src/Domain/Campaign/Campaign/CampaignService.php`

#### 1. Merge `module_config` instead of overwriting (`saveWithModules` + `updateWithModules`)

**Before:**
```php
if (isset($moduleData['module_config'])) {
    $campaignModule->setModuleConfig($moduleData['module_config']);
}
```

**After:**
```php
if (isset($moduleData['module_config'])) {
    $existingCmConfig = $campaignModule->getModuleConfig() ?? [];
    if (!empty($existingCmConfig)) {
        $campaignModule->setModuleConfig(
            $this->mergeModuleConfig($existingCmConfig, $moduleData['module_config'])
        );
    } else {
        $campaignModule->setModuleConfig($moduleData['module_config']);
    }
}
```

This uses the existing `mergeModuleConfig()` deep-merge utility (already used for `CampaignPhaseConfig`) to preserve runtime state keys like `start_date`/`end_date` while allowing new config values to override.

#### 2. Protect active/completed phase dates and config (`saveWithModules` + `updateWithModules`)

Added active-phase detection before the phase config update section:

```php
$isPhaseActive = $pcStart && $pcStart <= $now && (!$pcEnd || $pcEnd > $now);
$isPhaseCompleted = $pcEnd && $pcEnd < $now && $pcStart;
```

- **Date protection**: `start_date`/`end_date` on `CampaignPhaseConfig` are only updated when the phase is NOT active or completed.
- **Config date preservation**: For active/completed phases, the merged `module_config` preserves the existing `start_date` and `end_date` keys from the database instead of allowing stale request values to overwrite them.

#### 3. Fix Final Enrollment status lookup (`mapCampaignToDto`)

**Before:**
```php
$phasePrevious = $campaign->getCurrentActivePhaseDetails($module->getId() - 1);
```

**After:**
```php
if ($previousModule) {
    $phasePrevious = $campaign->getCurrentActivePhaseDetails($previousModule->getId());
}
```

Uses the already-computed `$previousModule` (found by `sequenceOrder - 1`) instead of fragile ID arithmetic.

#### 4. Module ID enrichment for `updateWithModules`

Pre-processes incoming module data to resolve `CampaignModule` IDs when the frontend sends modules without an explicit `id` but with `module_id` + `bidding_name`:

```php
// Enrich modules with IDs if they are missing but match existing modules by module_id and bidding_name
foreach ($modules as &$moduleData) {
    if (empty($moduleData['id']) && isset($moduleData['module_id'], $moduleData['bidding_name'])) {
        foreach ($campaign->getCampaignModules() as $cm) {
            if (!$cm->isDeleted() &&
                $cm->getModule()->getId() === (int)$moduleData['module_id'] &&
                $cm->getBiddingName() === $moduleData['bidding_name']) {
                $moduleData['id'] = $cm->getId();
                break;
            }
        }
    }
}
unset($moduleData); // prevent reference issues
```

Without this enrichment, existing modules sent without an `id` would be treated as **new** modules — creating duplicates and orphaning the original campaign module (along with its phase configs and execution records). This ensures that modules are correctly matched to their existing database records, so the merge and phase-protection logic in changes #1 and #2 can operate on the correct entities.

## Files Changed

| File | Change |
|------|--------|
| `src/Domain/Campaign/Campaign/CampaignService.php` | Merge `module_config` on `CampaignModule` instead of overwriting; protect active/completed phase dates and config from being reset; fix Final Enrollment previous-module lookup to use sequence order. |

## Impact

- **Bidding Stability**: Active bidding phases maintain their exact state and persisted data (submitted bids, statistics) even when generic campaign metadata is adjusted mid-cycle.
- **Correct Progression**: The Simulation phase boundary is protected. Completing a bidding round routes correctly to Simulation without jumping to Final Enrollment.
- **Backward Compatibility**: No schema modifications. No API contract changes. Updates operate entirely within the phase persistence layer. The `mergeModuleConfig()` utility was already in use — this change extends its usage to `CampaignModule` entities.

## Testing

- [ ] Start a Bidding phase, place bids (impersonating a student), edit main campaign min/max points in the PM dashboard → Bidding phase remains "Active" and bids are counted correctly.
- [ ] Close the Bidding phase → Simulation phase correctly opens and waits for user action (no jump to Final Enrollment).
- [ ] Verify Add/Drop phase statistics correctly reflect bids and enrollments from previous phases (no zero display).
