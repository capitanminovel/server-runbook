# Pre-Rolls Falsely Showed "New" (South Metro + Dinky Dope Menus) — Product Key Rewritten to Fix

**Date:** 2026-07-15
**Repos:** `mnlegitdev/legit-cannabis-south-metro-menu`, `mnlegitdev/dinky-dope-menu`

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

**Manual backdate:** the fix stops *future* renames from resetting `first_seen`, but the already-affected products carried forward whatever `first_seen` was on record at deploy time — which for the renamed items was itself the incorrect, already-reset value from before the fix existed. One-time manual correction: matched each renamed product to its pre-rename legacy entry (by name-prefix + brand + category, disambiguated since `last_seen` alone is shared by every product active in a given scrape) and restored the true historical `first_seen`. Done only for the specific known-affected products, not a blanket rewrite.

### Also applied to `dinky-dope-menu`

Same codebase lineage, same bug: this repo's `product_key()` already included `category` in the hash (a fix for an earlier, different incident — `[[dinky-buddy-api-product-key-collision]]`), but was still vulnerable to the same name-based fragility. Found **live and currently active** on this repo too — `PR Fog Dog`, `PR Frodo Squeeze`, `PR Gummi Bears` (Northpark) had all been relabeled with a `3pk - (1 gr ea)` suffix on 2026-07-14, showing false "New" badges. Applied the identical fix (id-based key, migration bridge in `merge()` and `enrich_strains.py`) and the same manual backdate for the 3 affected products. Verified: 74/74 products migrated cleanly, 0 false "new", enrichment step completed in ~1 second (no wasted API spend this time, since the near-miss was caught before this deploy).

One repo-specific note: because `category` was already part of the *old* key here, the `elif category changed` branch in `merge()` was dead code (a category change always produced a different old-style key, so it was always caught by the `NEW` branch first, never reaching the `elif`). Switching to a pure id-based key makes that branch live again — it will now correctly fire if Sweed ever reclassifies a product's category while keeping the same id. Kept it rather than removing it as "dead code."

### Mirror-image bug: false "Sold Out" from orphaned pre-rename entries

After the key migration, each renamed product's *pre-rename* legacy entry (a different name text = a different old-style key, never touched by the migration bridge since that only matches on the *current* name) was left behind in `products.json`, still `in_stock: false` with a recent `last_seen` from right before the rename. `build_preview.py`'s Sold Out section (`SOLD_DAYS = 2`, same style of `last_seen`-age filter as the New section) picked these up and showed them as "Gone" even though the product never left — it was renamed and is still on the menu under its new id-keyed entry.

Confirmed live on both repos and deleted the specific orphans (verified by name-prefix + brand + category match, not a blanket sweep):
- south-metro: `PR Bumper OG`, `PR Frodo Squeeze`, `PR Fog Dog`, `PR Gummi Bears`, `PR Medusa 2pk`, `PR Glue 31 - 1gr`, `PR Blueberry Muffin Mini Pre-rolls`
- dinky-dope-menu: `PR Fog Dog`, `PR Gummi Bears`, `PR Frodo Squeeze`

Their `first_seen` had already been extracted and applied to the new entries during the earlier backdate, so nothing was lost by removing them. Genuinely sold-out unrelated products (e.g. a flower-category `Glue 31` at south-metro, `Super Lemon Haze`/`Sour Apple` vapes at dinky-dope-menu) were left untouched.

## What I learned

- A hash of user-facing text is not a stable identity key — any field a POS/CMS can freely edit will eventually break it. Prefer the source system's own primary key when one exists, even if it means auditing the raw payload for it.
- Changing a key scheme is never local to one file — grep for every other place that keys off the same scheme before deploying. Here it was `strains_enriched.json`; on a bigger system it could be several.
- `gh run watch` blocks past ~2 min; for anything that might run long (API-cost steps especially), poll step-by-step status in the background so a runaway step can be caught and cancelled before it commits.
- A key-scheme migration that only bridges *forward* (old key → new key when the fresh item matches) leaves the *old* identity's dead history sitting in the data store. Any other feature keyed off recency of that same store (here: a "sold out" section keyed off `last_seen`) can independently misfire on that leftover record. When migrating identity, check every recency/age-based view of the data, not just the one that motivated the fix.

## Follow-up

`legit-cannabis-south-metro-menu` and `dinky-dope-menu` are separate repos with **near-identical code** (confirmed post-fix: same logic, only cosmetic formatting differs) rather than a shared library — this bug had to be found, fixed, tested, and deployed twice, and the same will be true of the next `scraper.py`/`enrich_strains.py` bug. Worth a conversation with the owner about whether these should share a common package (or one repo generates the other) instead of being maintained as independent copies. Not acted on now — just flagging so it doesn't get rediscovered from scratch next time.

## Follow-up

None — fix is deployed and verified. Related: `[[dinky-buddy-api-product-key-collision]]` is the same underlying fragility (name-based product identity) manifesting as a different symptom (key collision merging two products) rather than false "new" flags.
