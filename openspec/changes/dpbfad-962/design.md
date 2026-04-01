## Context

The campaign card (`BoxCampaign` component) in the Admin Dashboard renders all bidding rounds and their phases as a horizontal flex row. When a campaign has many bidding rounds (e.g. 8+ Elective_recovery rounds as seen in production), the content overflows the card with no mechanism to scroll, causing phases at the right side to be clipped and inaccessible.

The relevant container is the `div` at line 171 in `bidding-admin/src/components/campaign/box-campaign.tsx`:

```tsx
<div className='flex gap-6'>
  {data?.campaign_modules?.map(...)}
</div>
```

No backend, API, or data model changes are involved.

## Goals / Non-Goals

**Goals:**
- Allow horizontal scrolling within the campaign modules row inside each campaign card.
- Keep the overall page layout, card outer structure, and other UI components unchanged.
- All phases of all bidding rounds must be reachable via scroll.

**Non-Goals:**
- Changing the visual layout or ordering of bidding rounds.
- Adding pagination or a "show more" mechanism.
- Modifying the student portal (`bidding-web`).
- Any backend or API changes.

## Decisions

**Decision: Use `overflow-x: auto` with `flex-nowrap` on the modules container div**

The existing `div className='flex gap-6'` wrapping the campaign modules already lays out items in a row. Adding `overflow-x: auto` makes it scrollable when the content overflows. Tailwind classes: add `overflow-x-auto` and `pb-2` (bottom padding so the scrollbar doesn't overlap content).

Alternative considered: wrapping in a fixed-height container — rejected because the number of phases per round varies and a fixed height would clip content vertically.

Alternative considered: moving to a carousel/slider component (e.g. Ant Design Carousel) — rejected as over-engineering for a simple layout fix, and carousels hide content rather than exposing it.

**Decision: Inline Tailwind classes only — no new component**

The change is a one-line Tailwind class addition to an existing `div`. No new component, no abstraction is needed.

## Risks / Trade-offs

- **Scrollbar appearance on non-overflow content**: When fewer rounds exist, `overflow-x: auto` shows no scrollbar — no visual change for typical campaigns. Risk: negligible.
- **Thin scrollbar on macOS/trackpad**: macOS hides scrollbars by default; users may not discover scroll unless they try. Mitigation: this matches standard browser behaviour across the app; no special indicator needed given this is an admin interface used by power users.

## Migration Plan

- Single file change in `bidding-admin`.
- No deployment coordination required; purely visual, no data dependencies.
- Rollback: revert the Tailwind class change.
