# Flower/Pre-Roll Products Mislabeled as "Vapes" (South Metro + Dinky Dope Menus)

**Date:** 2026-08-05
**Repos:** `mnlegitdev/legit-cannabis-south-metro-menu`, `mnlegitdev/dinky-dope-menu`

## What happened

Flower (and potentially pre-roll) products were showing up under the "Vapes" section of the menu on both live sites.

## Why

Both scrapers query Sweed's `GetProductList` API once per category (`flower`, `pre-roll`, `edibles`, `vapes` — separate POST per category id), across all three scrape strategies (direct API, Playwright fetch, page-navigation intercept). Each strategy funnels its response through a shared `_parse_sweed_response(data, force_category=cat_name)`.

`_normalize_sweed_product()` already reads each item's own real category straight from the API payload (`item["category"]["name"]`). But `_parse_sweed_response` then unconditionally overwrote that with `force_category` — the category *being queried for*, not the item's actual category:

```python
if force_category:
    p["category"] = force_category
```

Sweed's per-category response isn't guaranteed to contain only items of that category (cross-sell widgets, imperfect server-side filtering can leak other items in). So when a flower item leaked into the response for the `vapes` category query, it got force-relabeled `"vapes"` — permanently, since nothing downstream re-derives category from the item itself.

Made worse by the merge/dedup step: `product_key()` keys products by Sweed's stable id only (`sw{id}`), with no category in the key. If the *same* product leaked into more than one category's response, whichever category was processed last (dict order puts `vapes` last) silently won on the collision, overwriting a correct entry.

## Fix

In `_parse_sweed_response`, only fall back to `force_category` when the item's own reported category is missing or not a recognized target category — a valid self-reported category is now always trusted over the query bucket:

```python
if force_category and p["category"].lower() not in TARGET_CATS:
    p["category"] = force_category
```

This fixes it at the source, so the merge-collision risk is also moot — any duplicate now carries the same correct category from either response.

Verified with a simulated `vapes`-category query response containing a leaked flower item: the flower item now stays `flower`, the genuine vape item stays `vapes`.

### Also applied to `dinky-dope-menu`

Same codebase lineage (see `[[2026-07-15-preroll-false-new-badge-product-key-rewrite]]`), byte-for-byte identical `_parse_sweed_response`. Same fix applied.

Both repos: committed to `scraper.py`, rebased onto the automated `🌿 Menu update` bot commits from the scheduled scrape runs, and pushed to `main`. Fix takes effect on the next scheduled/manual scrape — `docs/products.json` on both repos still reflects the old mislabeled data from before this fix, since GitHub Actions runs the scrape independently of this server; no local backfill was done here.

## What I learned

- A per-category API query is not a guarantee that every item in the response actually belongs to that category — trust the item's own fields over the bucket you queried under.
- `force_category`-style fallback parameters are a common source of this bug shape: they're meant as a *fallback* for missing/malformed data but get written as an unconditional override instead.

## Follow-up

None outstanding for this bug. Standing follow-up from the earlier incident still applies: these two repos are separate near-identical copies rather than a shared package, so bugs like this keep needing to be found and fixed twice.
