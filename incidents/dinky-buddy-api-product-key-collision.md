# Incident: Glue 31 Flower Missing from Menu (Product Key Collision)

**Date:** 2026-06-24  
**Repo:** `dinky-buddy-api` (same pattern applies to `legit-buddy-api`)

---

## What Happened

"Glue 31" flower was not appearing in the Dinky Dope menu even though the store carried it in Sweed. The pre-roll version of Glue 31 (same name, same brand) was showing correctly.

## Why

`product_key()` in `scraper.py` hashed only `name + brand`:

```python
key = f"{_normalize(p.get('name',''))}-{_normalize(p.get('brand',''))}"
return hashlib.md5(key.encode()).hexdigest()[:12]
```

Sweed has two separate SKUs for Glue 31 (Grasslandz):
- Flower — category ID 6450
- Pre-roll — category ID 6451

The scraper loops through `SWEED_CATEGORIES` in order and writes each result as `all_products[product_key(p)] = p`. Since flower is scraped first and pre-roll second, the pre-roll silently overwrites the flower entry. The flower product never made it into `products.json`.

This is a silent data loss — no error, no warning. The menu just shows one product instead of two.

## Fix

Include `category` in the key so same-name products in different form factors get distinct keys:

```python
def product_key(p: dict) -> str:
    cat = _normalize(p.get('category', ''))
    key = f"{_normalize(p.get('name',''))}-{_normalize(p.get('brand',''))}-{cat}"
    return hashlib.md5(key.encode()).hexdigest()[:12]
```

Cross-store enrichment sharing is unaffected — the same product at two stores has the same name, brand, AND category, so keys still match across stores.

### Migration

Changing the key scheme orphans all existing entries in `products.json` and `strains_enriched.json`. Both files must be re-keyed in place before the next build. Pattern:

```python
import json, hashlib, unicodedata

def _normalize(s):
    s = unicodedata.normalize("NFD", s).encode("ascii", "ignore").decode().lower().strip()
    return s.rstrip(" -–—.,")

def new_key(name, brand, category):
    key = f"{_normalize(name)}-{_normalize(brand)}-{_normalize(category)}"
    return hashlib.md5(key.encode()).hexdigest()[:12]

# Re-key products.json
with open("docs/products.json", encoding="utf-8") as f:
    db = json.load(f)
new_products = {}
for old_k, p in db["products"].items():
    new_products[new_key(p["name"], p["brand"], p["category"])] = p
db["products"] = new_products
with open("docs/products.json", "w", encoding="utf-8") as f:
    json.dump(db, f, indent=2, ensure_ascii=False)

# Re-key strains_enriched.json (needs cross-ref with products.json to know category)
with open("docs/strains_enriched.json", encoding="utf-8") as f:
    enriched = json.load(f)
migrated = {}
for old_k, data in enriched.items():
    p = db["products"].get(old_k)  # look up by old key before re-keying products
    if p:
        migrated[new_key(p["name"], p["brand"], p["category"])] = data
    else:
        migrated[old_k] = data  # orphaned (OOS/removed) — keep as-is
with open("docs/strains_enriched.json", "w", encoding="utf-8") as f:
    json.dump(migrated, f, indent=2, ensure_ascii=False)
```

**Important:** re-key `strains_enriched.json` BEFORE re-keying `products.json`, or do the migration in one script that holds both in memory simultaneously (as done here).

## What I Learned

- Same-name products in different form factors (flower vs pre-roll, 1g vs 2-pack) can share a `name + brand` key. Always include a differentiator.
- The `all_products[key] = p` overwrite pattern fails silently — there's no collision warning. Worth adding a `log()` call for collisions in the future.
- Re-keying existing data is safe as long as you can look up the category from the product record. All 47 entries migrated cleanly with zero orphans.

## Follow-Up

- Glue 31 flower will appear automatically on the next scheduled scrape (runs 4x daily via GitHub Actions). `enrich_strains.py` will enrich it in the same run.
- Consider applying the same `category`-in-key fix to `legit-buddy-api` / `legit-buddy-private` if they share the same `product_key()` pattern.
