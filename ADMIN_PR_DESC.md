# Campaign Dashboard — Add Scroll Bar for Campaign Card

## Problem
The campaign card in the Admin Dashboard renders all bidding rounds and phases in a horizontal flex row. When a campaign has many rounds (e.g. 7+ Elective_recovery rounds), the content overflows the card boundary with no scroll mechanism, making right-side phases inaccessible.

## Goal
Add horizontal scrolling to the campaign modules row so that all bidding rounds and their phases are accessible regardless of how many rounds a campaign has.

## Changes Made

1. **`src/components/campaign/box-campaign.tsx`**
   - Added `overflow-x-auto pb-2` to the campaign modules container div, enabling horizontal scroll when content exceeds the card width. The `pb-2` provides spacing so the scrollbar doesn't overlap phase labels.

## Testing / Verification Steps
1. Open the Admin Dashboard with a campaign that has 7+ bidding rounds — confirm a horizontal scrollbar appears on the phases row and all rounds are reachable by scrolling.
2. Confirm that campaigns with few bidding rounds show no scrollbar and the layout is unchanged.
3. Confirm clicking a phase still navigates to the detail-bidding page correctly.
4. Confirm the card header, stats boxes, dates, and overall page layout are unaffected.
