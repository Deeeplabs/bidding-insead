## 1. Frontend — Add Horizontal Scroll to Campaign Modules Row

- [x] 1.1 In `bidding-admin/src/components/campaign/box-campaign.tsx`, update the campaign modules container div (line 171) by adding `overflow-x-auto pb-2` to its Tailwind classes — changing `className='flex gap-6'` to `className='flex gap-6 overflow-x-auto pb-2'`

## 2. Verification

- [x] 2.1 Open the Admin Dashboard with a campaign that has 7+ bidding rounds and confirm a horizontal scrollbar appears on the phases row and all rounds are reachable by scrolling
- [x] 2.2 Confirm that campaigns with few bidding rounds show no scrollbar and the layout is unchanged
- [x] 2.3 Confirm clicking a phase still navigates to the detail-bidding page correctly
- [x] 2.4 Confirm the card header, stats boxes, and page layout are unaffected
