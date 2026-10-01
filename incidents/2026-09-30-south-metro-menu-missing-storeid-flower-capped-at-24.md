# South Metro live menu: 7 in-stock flower items missing, false "Sold Out" entries

**Date:** 2026-09-30
**Repos:** `mnlegitdev/legit-cannabis-south-metro-menu` (live) and `capitanminovel/legit-buddy-api` (same bug, same output)
**Status:** fixed 2026-10-01 in both mnlegitdev repos (south-metro `365d8af`, dinky-dope `f775fdb`); `capitanminovel/legit-buddy-api` still has the bug (left alone on purpose)

## What happened
The user noticed that out-of-stock status on the South Metro menu didn't match the store. We compared it to a direct pull from Sweed (`storeid: 434`, all pages):

| Category | Live on Sweed | Shown in stock on site |
|---|---|---|
| Flower | 31 | 24 |
| Pre-roll | 10 | 10 |
| Edibles | 4 | 4 |
| Vapes | 0 | 0 |

The site was missing 7 flower items that are in stock: Gorilla Zkittlez, Grape Canyon Zkittles, Magic Marker, Glitter Bomb, Hippie Crasher (all Float), Dante's Inferno (Campfire Cannabis) and Double Sour Grape (Tasty Gems). Double Sour Grape was also listed on the **Sold Out** tab even though it was in stock.

Nothing was wrongly shown as in stock. The two pre-rolls on the Sold Out tab (PR Hyperion F1 2pk, Donny Burger) really are gone from Sweed.

Git history shows in-stock flower sitting at **exactly 24 in every scrape for weeks**, which points to a hard cap rather than real inventory.

## Why
1. South Metro's `scraper.py` `HEADERS` has **no `storeid` header**. Sweed returns HTTP 400 without it (tested: no header gives 400, `storeid: 434` gives 200). The scraper treats that as "WAF blocked" and falls back to Playwright.
   - Dinky Dope's scraper sends `storeid: 609`, so its direct API works and pages through correctly (flower p1=24, p2=3).
2. The Playwright fallback (`try_playwright`) loads each category page and captures only the **first** `GetProductList` response the page makes, which holds 24 items. The page loads the rest on scroll, and the scraper never scrolls, so anything past item 24 is never seen.
3. `merge()` marks anything not seen as `in_stock=False`. Items on page 2 are therefore treated as sold out. Sweed's sort order shifts between scrapes, so an item can move from page 1 to page 2 and flip to a false "Sold Out", which is what happened to Double Sour Grape.

## Fix (applied to mnlegitdev repos)
Add the same store headers Dinky Dope uses to South Metro's `scraper.py`:
```python
STORE_ID = 434
# ... same _last_store cookie block as dinky-dope-menu ...
HEADERS = { ..., "storeid": str(STORE_ID), "Origin": ..., "ssr": "false", "Cookie": ... }
```
With this, the direct API succeeds and the existing pagination loop fetches all 31 flower items. Playwright only runs if Sweed actually blocks us.

Apply the same change to `capitanminovel/legit-buddy-api`, which has the same output (38 products via Playwright).

Secondary hardening (optional):
- In `try_sweed_api`, stop paging when `len(raw_items) < page_size` or when `page * size >= total`, not based on the count **after** category filtering.
- Add a sanity guard: if a scrape returns far fewer products than are currently in stock, don't mark the missing ones as out of stock.

## What I learned
- Sweed requires `storeid` as an HTTP header. We learned this for dispo_menu on 2026-08-26, but it had never been ported back to South Metro.
- If a count stays at a round number (24) across weeks of scrapes, suspect a page-size cap before believing the inventory.
- The "Direct API blocked" log line was misleading here. It was a 400 from a missing header, not a WAF block.

## Also done in the same change (both mnlegitdev repos)
- Added the **Concentrates** category (Sweed id 5251 for South Metro, 6453 for Dinky Dope) to `TARGET_CATS`, `SWEED_CATEGORIES`, `_CAT_NORM`, `CATEGORY_PAGE_URLS`, the `build_preview.py` `TARGET` list, the dark-mode section accent color and the guide text. Right now that's 3 Marawanna live rosins at each store.
- Pagination now stops based on the raw page size Sweed returns, not the count after category filtering.
- Verified with manual workflow runs. South Metro now shows 48 in stock (31 flower, 10 pre-roll, 4 edibles, 3 concentrates), and Dinky Dope shows 56. Both match a direct pull from Sweed. Dinky Dope's stock was already correct before this change (53 = 53).
- The 6 recovered flower items show a "New" badge for 3 days, because the site had never recorded them before.

## Follow-up
- [ ] `capitanminovel/legit-buddy-api` still has the missing-storeid bug. The user said to fix only the mnlegitdev repos.
- [ ] The menus still exclude Hemp Derived THC Products, Wellness and Accessories. That's a deliberate product decision, so it's still open.
- [ ] The terpene-order-as-dominance prompt bug in `enrich_strains.py` is still present in both mnlegitdev repos. The new concentrate profiles were generated with it.
