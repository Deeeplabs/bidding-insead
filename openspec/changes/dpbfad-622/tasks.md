## 1. Backend Implementation - API Authorization

- [x] 1.1 Find the CampaignController that serves pre-bidding data (likely in `bidding-api/src/Controller/Api/Student/Campaign/`)
- [x] 1.2 Add Programme/Promotion authorization check before returning campaign data
- [x] 1.3 Return 403 Forbidden if user's Promotion doesn't match Campaign's Promotion
- [x] 1.4 Handle edge case where Campaign has no Programme assigned (allow access for backward compatibility)

## 2. Frontend Implementation - bidding-web

- [x] 2.1 Find the pre-bidding page component in `bidding-web/src/features/bidding/pages/`
- [x] 2.2 Add authorization validation on page mount (check user's Programme/Promotion against campaign)
- [x] 2.3 Redirect to Access Denied page if authorization fails
- [x] 2.4 Ensure no campaign data is rendered before authorization check

## 3. Verification & Testing

- [ ] 3.1 Test authorized user accessing matching Programme/Promotion campaign (should succeed)
- [ ] 3.2 Test unauthorized user accessing different Programme campaign via direct URL (should redirect to Access Denied)
- [ ] 3.3 Test API endpoint returns 403 for unauthorized access
- [ ] 3.4 Verify no campaign data exposed in unauthorized response
- [ ] 3.5 Smoke test: verify existing campaigns still work for authorized users
