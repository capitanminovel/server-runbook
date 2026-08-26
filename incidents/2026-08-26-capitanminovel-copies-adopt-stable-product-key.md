# capitanminovel/legit-buddy-api + dinky-buddy-api Never Got the 2026-07-15 Stable-Key Fix

**Date:** 2026-08-26
**Repos:** `capitanminovel/legit-buddy-api`, `capitanminovel/dinky-buddy-api`

## What happened

Same day as `[[2026-08-26-new-arrivals-first-seen-never-reset-on-restock]]`: after porting the restock/`first_seen` fix to the two `capitanminovel` "stale" copies (which turned out to be fully live — see `[[project_mnlegitdev_repo_ownership]]`), reading their `scraper.py` history surfaced a second, older gap: neither repo ever adopted the `2026-07-15` fix (`[[2026-07-15-preroll-false-new-badge-product-key-rewrite]]`) that switched the mnlegitdev repos' `product_key()` from a text-hash of name+brand to Sweed's own stable `id`. Both capitanminovel copies still hashed name+brand only — the exact scheme that made a POS-side rename/relabel look like a brand-new product (false "New Today" badge) and orphan the old entry (false "Sold Out" too).

Confirmed via a git-log dig in `legit-buddy-api`: its own commit history is full of ad-hoc attempts at this same class of problem over months — `Fix new-badge stale data and brand rename key collisions`, `Fix Avió duplicate keys from normalize experiment`, `Backdate Avió first_seen`, `Fix GG4 pre-roll new badge: reset first_seen` — none of which actually adopted a stable identity key; they kept patching symptoms of the same root cause the mnlegitdev repos fixed properly once.

## Why

Neither repo's `_normalize_sweed_product()` ever read Sweed's `raw.get("id")` — it was available in every raw API response the whole time (same as the mnlegitdev repos, confirmed in the original `07-15` incident) but simply never captured.

## Fix

Ported the `07-15` fix to both repos:
- `scraper.py`: capture `"sweed_id": raw.get("id")` in the normalizer. Renamed the existing name+brand hash function to `_legacy_key()` (kept `dinky-buddy-api`'s existing category-inclusion as-is). New `product_key()` prefers `sw{id}`, falling back to `_legacy_key()` only for DOM-scrape-fallback items with no id. `merge()` bridges a legacy-keyed record onto its new id-based key on first sight, preserving `first_seen` — combined with the same-day `RESTOCK_GAP_DAYS` check so a migration that also happens to coincide with a restock resets correctly instead of silently preserving stale history.
- `enrich_strains.py`: added the identical migration bridge before the new-product enrichment step in both repos. This is the step that actually carries risk — the original `07-15` fix's first deploy attempt forgot this exact bridge and the entire already-enriched menu suddenly looked "new" to the AI enrichment script, which started burning real Claude API calls re-enriching everything before it was caught mid-run and cancelled.

**Verification before deploying (learned from that near-miss, not repeating it):**
1. Wrote a synthetic full-history merge dry run for each repo (assign every currently-tracked product a fake `sweed_id`, run it through the new `merge()`, confirm every product's `first_seen` is preserved from its legacy record rather than reset). Results: `legit-buddy-api` 146/148 clean (2 pre-existing orphaned duplicate records from its own past ad-hoc key experiments don't match cleanly — see below); `dinky-buddy-api` 121/121 clean, zero mismatches.
2. After pushing, manually triggered a real `workflow_dispatch` run on both repos instead of waiting for the next cron, and polled step-by-step status rather than using `gh run watch` (which blocks past ~2 min) so a runaway `enrich_strains.py` step could be caught and cancelled before it committed — same caution as the original incident.
3. Confirmed on both real runs: `legit-buddy-api` migrated 32 active products (its currently-live subset) cleanly, log showed `All products already enriched — skipping profile step.` — zero re-enrichment API calls. `dinky-buddy-api` migrated 55, same zero-re-enrichment result. Pulled the resulting commits and confirmed New Arrivals counts were unaffected (6 and 7 respectively, same as before the key-scheme change) and both repos' data now carry real `sweed_id` values on their active products.

Commits: `0a02391` (legit-buddy-api scraper/enrich), triggered run `33022847361`; `78b67e1` (dinky-buddy-api scraper/enrich), triggered run `33022971817`.

**Known pre-existing issue, not fixed here:** `legit-buddy-api`'s dry run flagged 2 products (`Blueberry Muffin`/Grasslandz and a "Blueberry Muffin Mini Pre-rolls-" variant) whose *currently stored* legacy key doesn't match a fresh recomputation of their own current name+brand fields — i.e. these are already-orphaned duplicate records left over from this repo's own pre-`07-15`-equivalent, ad-hoc key-scheme churn (evidenced by the commit history above). They'll each re-enrich/re-date once on their next real appearance in a scrape, then track correctly forever after under their new id key. Left alone rather than chasing down every historical orphan — same judgment call as the `07-15` incident's "orphaned pre-rename entries," but scoped down since these are already `in_stock: false` and invisible either way.

## What I learned

- When porting a fix to a sibling/copy repo, check whether it diverged from *before* the bug being ported was even introduced — this repo's history showed it had already tried to solve the same problem several times its own way (badly) rather than just missing one clean fix.
- The safest way to validate a key-scheme migration before it touches a real Anthropic API key: simulate the full migration locally with synthetic data first (catches structural bugs for free), *then* trigger one real `workflow_dispatch` run and poll it step-by-step rather than trusting the next unattended cron to go well.

## Follow-up

None outstanding for this specific fix. Standing note from `[[project_mnlegitdev_repo_ownership]]` still applies: four independently-maintained scraper codebases against two physical stores means every `scraper.py`/`enrich_strains.py` bug now needs checking in up to four places.
