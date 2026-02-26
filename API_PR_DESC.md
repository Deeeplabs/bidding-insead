# Fix PM Dashboard Active Campaign Count Mismatch

## Problem

The Programme Manager Dashboard displays an incorrect count for "Active Campaigns" in two places:

1. **Header Stats Widget** — Shows "Active Campaigns: 68 / Running now", which is higher than the actual number of currently active campaigns.
2. **Pagination Label** — Shows "Showing 61 - 68 of 68 campaigns" on the last page, but the table only renders 4 items (not 8).

The visual count in the table and the total in the header/pagination do not match after manual verification.

### Root Cause

The mismatch is caused by missing `endDate >= NOW()` filtering in two queries that compute **totals**, while the query that fetches the **paginated rows** already includes this filter:

| Query | Location | Had `endDate >= NOW()`? |
|-------|----------|------------------------|
| Data fetch (paginated rows) | `CampaignService::listCampaigns` — `$qb` | ✅ Yes |
| Count query (pagination meta) | `CampaignService::listCampaigns` — `$countQb` | ❌ **No** |
| Dashboard stats (header widget) | `CampaignRepository::findOpenCampaigns` | ❌ **No** |

Campaigns whose `endDate` has passed but whose `status` was never changed from `'open'` were excluded from the paginated data but still counted in the total — inflating both the header stat and the pagination metadata.

## Solution

### Changes Made

#### 1. Fix pagination count query — `CampaignService::listCampaigns`

**File:** `src/Domain/Campaign/Campaign/CampaignService.php`

Added the `endDate >= :now` condition to `$countQb` when `$status === 'open'`, mirroring the existing condition on the data query `$qb`:

```php
if ($status) {
    if ($status === 'open') {
        $countQb->andWhere('c.endDate >= :now')
                ->setParameter('now', new \DateTime());
    }
    $countQb->andWhere('c.status = :status')
            ->setParameter('status', $status);
}
```

This ensures `meta.total` and `meta.total_pages` accurately reflect the number of campaigns actually returned by the paginated data query.

#### 2. Fix dashboard stats query — `CampaignRepository::findOpenCampaigns`

**File:** `src/Repository/CampaignRepository.php`

Added the `endDate >= :now` condition to the query used by `PMDashboardStatsService` for the header widget:

```php
return $this->createQueryBuilder('pc')
    ->where('pc.program IN (:programIds)')
    ->setParameter('programIds', $programIds)
    ->andWhere('pc.status = :status')
    ->setParameter('status', 'open')
    ->andWhere('pc.endDate >= :now')          // ← added
    ->setParameter('now', new \DateTime())    // ← added
    ->andWhere('pc.isDeleted = :isDeleted')
    ->setParameter('isDeleted', false)
    ->getQuery()
    ->getResult();
```

This ensures the "Active Campaigns: N / Running now" header stat only counts campaigns that are truly still running.

## Files Changed

| File | Change |
|------|--------|
| `src/Domain/Campaign/Campaign/CampaignService.php` | Add `endDate >= :now` filter to the count query builder (`$countQb`) when filtering by `status = 'open'`, aligning it with the data fetch query. |
| `src/Repository/CampaignRepository.php` | Add `endDate >= :now` filter to `findOpenCampaigns()` used by the PM Dashboard header stats. |

## Impact

- **Dashboard Header**: "Active Campaigns" count now accurately reflects only campaigns with `status = 'open'` AND `endDate >= NOW()`.
- **Pagination**: `meta.total` matches the actual number of rows returned, so "Showing X - Y of Z campaigns" is correct on every page.
- **Backward Compatibility**: No schema changes. No API response shape changes. Purely a query filter alignment fix.
- **Risk**: Campaigns whose `endDate` has silently passed will no longer appear in active counts. This is the correct behavior — they were already excluded from the paginated list.

## Testing

- [ ] Navigate to the PM Dashboard → Verify the "Active Campaigns" count in the header matches the total number of campaigns visible when paging through the list.
- [ ] Go to the last page of the active campaigns table → Verify the pagination label ("Showing X - Y of Z") matches the actual number of rows displayed.
- [ ] Check that campaigns with a past `endDate` but `status = 'open'` are excluded from both the header count and pagination total.
