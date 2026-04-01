## Why

The campaign card in the Admin Dashboard renders all bidding rounds and their phases in a horizontal flex row. When a campaign has many rounds (e.g. 7+ Elective_recovery rounds), the content overflows the card with no scroll mechanism, truncating phases and making them inaccessible to the programme manager.

## What Changes

- Add `overflow-x: auto` (horizontal scroll) to the campaign module phases row inside the `BoxCampaign` component.
- The scrollable area is limited to the phases row within the card — the rest of the card layout, page layout, and other UI components remain unchanged.

## Capabilities

### New Capabilities

- `campaign-card-scroll`: Horizontal scrollbar on the campaign module phases row in `BoxCampaign`, enabling access to all bidding rounds when content exceeds visible width.

### Modified Capabilities

<!-- No existing spec-level requirements are changing. -->

## Impact

- **bidding-admin only** — pure frontend CSS/layout change.
- **File**: `bidding-admin/src/components/campaign/box-campaign.tsx` — the `div` at line 171 wrapping `data.campaign_modules.map(...)` needs `overflow-x: auto` and optionally `pb-2` for scrollbar spacing.
- No API changes, no entity changes, no migrations, no backend changes.
- No breaking changes.
