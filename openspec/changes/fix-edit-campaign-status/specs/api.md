# API Specification: Fix Campaign Status Reversion

## Overview
This specification outlines the backend API adjustments necessary to resolve the desynchronization bug where a campaign phase forcefully reverts to its draft boundaries or abruptly closes when other campaign modules are updated. 

---

## 1. Activate Campaign Phase

**Endpoint:** `PUT /v2/api/dashboard/{campaignId}/{moduleCode}/activate-phase`

**Purpose:** Triggers the start (`open`) or end (`close`) of a selected module execution round within a campaign.

**Request Payload:**
```json
{
  "phase_id": 67,
  "status": "open" // or "close"
}
```

**Implementation Changes:**
The service logic inside `CampaignPhaseService::activatePhase()` evaluates both states. We maintain synchronization directly within this controller action path:

*   **When opening a phase (`status: "open"`):**
    *   Sets `$now` onto the scalar column `start_date` within the `CampaignPhaseConfig` associated entity.
    *   **NEW:** Deserializes the JSON attribute `module_config`, attaches/replaces `['start_date'] => $now->format(\DateTime::ATOM)`, and reserializes it into the entity.

*   **When closing a phase (`status: "close"`):**
    *   Sets `$now` onto the scalar column `end_date` within the `CampaignPhaseConfig` associated entity.
    *   **NEW:** Deserializes the JSON attribute `module_config`, attaches/replaces `['end_date'] => $now->format(\DateTime::ATOM)`, and reserializes it into the entity.

**Response (Success - 200 OK):**
```json
{
  "start_date": "2024-03-24 15:00:00",
  "end_date": "2024-03-28 15:00:00",
  "status": "open"
}
```

---

## 2. Update Campaign With Modules Bulk Edit

**Endpoint:** `PUT /v2/api/campaigns/{id}/save-with-modules`

**Purpose:** Allows administrators to bulk-upsert or reorder campaign configurations safely. Includes all module settings, configurations, constraints, and JSON `module_config` maps.

**Implementation Changes:**
*   **Validation & Constraints:** No direct code modifications are applied to this controller mapping function.
*   **Safety Implication (Fix):** Because `activate-phase` properly normalizes the JSON data blob within the database, `GET /v2/api/campaigns/{id}` consistently delivers the accurately updated dates to the client frontend. Next time `save-with-modules` pushes its update with the unedited modules, the payload securely matches the exact timestamp limits. This eliminates the edge case where the active phase becomes completely overwritten with previous legacy draft data and immediately ends the round.
