## Context

Currently, students can access pre-bidding URLs for campaigns by copying/pasting the URL directly. The system validates programme and promotion authorization only during UI navigation (e.g., when clicking through from the dashboard), but not when accessing the URL directly. This allows a student assigned to EMBA to view MBA campaign details by navigating to the MBA pre-bidding URL.

The issue is:
1. The frontend (bidding-web) checks authorization only on initial load via navigation
2. The backend (bidding-api) returns campaign data without verifying Programme/Promotion match
3. No protection against direct URL access

## Goals / Non-Goals

**Goals:**
- Validate Programme and Promotion authorization on every pre-bidding page access
- Redirect unauthorized users to Access Denied page
- Ensure no campaign data is exposed before authorization check
- Maintain backward compatibility for authorized users

**Non-Goals:**
- Change campaign phase validation logic (existing behavior)
- Modify how campaigns are assigned to Programme/Promotion in admin
- Add new database fields (this is a logic change only)

## Decisions

### 1. Backend Validation Strategy
**Decision**: Add authorization check in the CampaignController endpoint that serves pre-bidding data.

**Alternative considered**: Create a new middleware - Rejected because existing controller pattern is consistent with other authorization checks in the system.

**Implementation**: 
- In `CampaignController` (or whichever serves `/api/student/campaign/{id}/pre-bidding`), add a check that compares:
  - User's Promotion (from authenticated session)
  - Campaign's Promotion (from Campaign entity)
- Return 403 Forbidden if they don't match

### 2. Frontend Validation Strategy
**Decision**: Add authorization validation on pre-bidding page mount in bidding-web.

**Alternative considered**: Rely solely on backend - Rejected because we need to show a proper "Access Denied" page, not just an error.

**Implementation**:
- In the pre-bidding page component (`src/features/bidding/pages/pre-bidding.tsx` or similar):
  - On mount, check if user's Programme/Promotion matches campaign's Programme/Promotion
  - If not authorized, redirect to Access Denied page
  - Use the existing user context to get student's promotion

### 3. Access Denied Page
**Decision**: Reuse existing Access Denied page rather than creating a new one.

**Implementation**:
- The system likely has an existing access-denied or unauthorized page
- Redirect unauthorized users to that page with appropriate message

## Risks / Trade-offs

### Risk: Race condition between frontend and backend validation
**Mitigation**: Primary validation is on backend; frontend validation is for UX (proper redirect). Backend always enforces authorization.

### Risk: Breaking existing behavior for edge cases
**Mitigation**: Only add the check for Programme/Promotion mismatch; if either is null/unset, allow access (maintaining backward compatibility for legacy data).

### Risk: Performance impact from additional query
**Mitigation**: The campaign is already loaded; checking promotion is a simple property comparison (O(1)).

### Trade-off: Dual validation (frontend + backend)
**Justification**: Backend is authoritative; frontend provides better UX with proper redirect. Both are needed for security + UX.
