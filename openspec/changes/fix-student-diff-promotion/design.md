## Context

Currently, a student who is manually added to a campaign for the "Pre-bidding ONLY" configuration is not able to see the campaign on their dashboard or bidding page, especially if they are from a different promotion. When a campaign is created and the student is impersonated, they see all bidding phases including Add/Drop and Waitlist, even though they should only have access to Pre-bidding. Manual student addition needs to be properly constrained to only the pre-bidding configuration, and the system needs to accurately filter out other bidding phases when a student is only configured to participate in one specific round.

## Goals / Non-Goals

**Goals:**
- Fix the logic that hides the campaign on the dashboard/bidding page for students added manually to the "Pre-bidding ONLY" round, regardless of their native promotion.
- Ensure that manual student addition is only available or applied during the pre-bidding configuration.
- Adjust the student bidding view (and impersonation view) to strictly display only the bidding rounds the student is individually added to (e.g., hiding Add/Drop and Waitlist if they are added only for Pre-bidding).

**Non-Goals:**
- Completely rewriting the bidding system access controls.
- Changing how standard promotion-based assignments work for students.
- Refactoring the entire student dashboard beyond the scope of this visibility fix.

## Decisions

- Modify the database query or API filtering logic that provides the campaigns list for a student, ensuring it explicitly includes campaigns where the student was manually listed as a participant for Pre-bidding, bypassing strict promotion checks that might currently block them.
- Adjust the bidding round phase presentation logic to filter out rounds (like Add/Drop and Waitlist) if the current student is identified as a manual addition limited to a specific round (Pre-bidding).
- Enforce validation or logic rules in the admin campaign editing configurations that restrict manual student assignments strictly to pre-bidding participation.

## Risks / Trade-offs

- **Risk:** Altering the campaign visibility query could inadvertently expose campaigns to students who shouldn't see them if the manual addition flags aren't checked rigorously.
- **Risk:** Changing the participation round visibility logic might affect normal flow students if not scoped to only manually added students.
- **Trade-off:** Minimal scope is preferred to prevent breaking existing functionality, but filtering out other phases selectively based on manual participation adds slight complexity to the phase resolution logic.
