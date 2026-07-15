# 6 Pre-Rolls Falsely Showed "New" (South Metro Menu) — Product Key Rewritten to Fix

**Date:** 2026-07-15
**Repo:** `mnlegitdev/legit-cannabis-south-metro-menu`

## What happened

6-7 pre-rolls that had been in stock for weeks suddenly showed a "New Today" badge. All of them were Sweed POS relabels — e.g. `PR Bumper OG` → `PR Bumper OG 3pk (1 G ea)` — not actually new products.

## Why

`product_key()` in `scraper.py` derived a product's identity by hashing `normalize(name) + normalize(brand)`. Any POS-side text change (pack-size added to the name, punctuation dropped) produced a different hash, so `merge()` treated the relabeled product as brand new and reset `first_seen`.

Confirmed by pulling a real raw Sweed API product via a temporary GitHub Actions artifact upload (Strategy 1 direct-POST is WAF-blocked outside a real browser — only the Playwright path, which intercepts the live page's own API calls, sees genuine JSON). The raw payload has a stable `id` field (e.g. `"id": 373921`) and a matching `variants[].id` — both completely discarded by the parser, which only ever read nested `.name` fields (`brand.name`, `category.name`, etc.).

## Fix

- `product_key()` now returns `sw{id}` when Sweed's own `id` is present, falling back to the old text-hash only for DOM-scrape-fallback items that have no `id`.
- `merge()` migration bridge: if a fresh item's new id-based key isn't found, check the old text-based key — if found, carry over `first_seen` and delete the old key. One-time cutover; no history lost.
- Deployed and verified against live data: 31 active products migrated cleanly (0 false "new"), 72 historical/out-of-stock entries left untouched.

**Near-miss caught mid-deploy:** `strains_enriched.json` (AI-generated strain profiles) is *also* keyed by `product_key()`. The first deploy attempt rewrote the product key scheme but not the enrichment lookup — every one of the ~103 already-enriched products suddenly looked "new" to `enrich_strains.py`, which started re-enriching the entire menu via the Claude API. Caught ~6 minutes in (via `gh run watch` + step-status polling) and cancelled before it committed anything — but real API spend had already happened for whatever it processed in that window. Fixed by adding the same legacy-key migration bridge to `enrich_strains.py` before its `existing`/`new_keys` check.

## What I learned

- A hash of user-facing text is not a stable identity key — any field a POS/CMS can freely edit will eventually break it. Prefer the source system's own primary key when one exists, even if it means auditing the raw payload for it.
- Changing a key scheme is never local to one file — grep for every other place that keys off the same scheme before deploying. Here it was `strains_enriched.json`; on a bigger system it could be several.
- `gh run watch` blocks past ~2 min; for anything that might run long (API-cost steps especially), poll step-by-step status in the background so a runaway step can be caught and cancelled before it commits.

## Follow-up

None — fix is deployed and verified. Related: `[[dinky-buddy-api-product-key-collision]]` is the same underlying fragility (name-based product identity) manifesting as a different symptom (key collision merging two products) rather than false "new" flags.
