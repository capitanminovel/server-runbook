# Gummy Product Mistagged as "Vapes" at the Sweed Source (South Metro + Dinky Dope Menus)

**Date:** 2026-08-05
**Repos:** `mnlegitdev/legit-cannabis-south-metro-menu`, `mnlegitdev/dinky-dope-menu`

## What happened

"Watermelon - Hybrid" (Relax brand, `sweed_id: 226964`) showed up under the Vapes section on the South Metro live menu. Its description made clear it's actually an edible: "10 count / 5mg THC + 2mg CBC per Gummy ... Vegan & Gluten Free."

This is a different bug from the same-day `[[2026-08-05-flower-mislabeled-as-vapes-force-category]]` incident. That one was the scraper overwriting a correct self-reported category with the wrong query bucket. This one is different: the scraper was already trusting the item's own self-reported category (post-fix) — but Sweed itself reported the wrong category for this item. The item was brand new (`first_seen` == `last_seen`, added 2026-08-04) and also missing THC/CBD/terpene/effects data, consistent with a dispensary staff member still finishing setup of a new SKU in the POS and picking the wrong category dropdown.

## Why

We (capitanminovel) have collaborator access to the GitHub repos but **no access to the dispensary's Sweed POS admin** — we can't fix the category tag at the source. The scraper has no way to know the tag is wrong except by cross-checking other signals already in the payload.

## Fix

Added a targeted override in `_normalize_sweed_product()`: if the description contains "gummy"/"gummies" (dosage language like "per Gummy") and the parsed category isn't already `edibles`, force it to `edibles`. This runs after the existing self-reported-category normalization, so it only kicks in when Sweed's own tag disagrees with what the product description says.

```python
_GUMMY_RE = re.compile(r'\bgumm(?:y|ies)\b', re.I)

def _category_from_description(description: str, category: str) -> str:
    if category != "edibles" and _GUMMY_RE.search(description or ""):
        return "edibles"
    return category
```

Checked against the live South Metro dataset before shipping: only the one mistagged product (`sw226964`) matched, no other in-stock items got relabeled — so no evidence of false positives from this keyword on real data.

Manually corrected `docs/products.json` category for `sw226964` and re-ran `build_preview.py` on South Metro so the live site reflects the fix immediately rather than waiting for the next scheduled scrape (products.json is written by GitHub Actions independently of this server, so future scrapes will now apply the fix automatically). `dinky-dope-menu` had no currently mistagged product, so only `scraper.py` changed there — no data backfill needed.

Both repos: committed to `scraper.py` and pushed to `main`.

## What I learned

- Two distinct failure modes can produce the same visible symptom ("wrong category showing"): scraper-side override logic vs. wrong data at the source system. Worth checking the raw API response before assuming a scraper bug.
- When there's no write access to the upstream source of truth (Sweed here), the fallback is inferring from a field we *do* control the parsing of (description text) rather than trying to fix the source.

## Follow-up

Same standing item as the sibling incident: `legit-cannabis-south-metro-menu` and `dinky-dope-menu` are independent near-identical copies, not a shared package — this class of bug needs to be found and fixed twice until that's addressed.
