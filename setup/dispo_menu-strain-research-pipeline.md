# dispo_menu — Strain Generator research pipeline (2026-09-24)

## Why it changed
The first design gave Claude `web_search` + `web_fetch` and let it loop. Server-side tools re-send the
whole growing context every round trip, so one strain cost **~$1.15** (354k input tokens, ~20 round
trips) — and the cheaper Sonnet run was *thinner*, not just cheaper. Now **our code reads the pages
(free) and Claude makes one call (~5–15¢)**. The old loop remains as `deep=true` on the API (no UI yet).

## How it works (per generation)
1. `app/ai/research.py::gather_research` runs, in parallel:
   - **Page fetcher** (`app/ai/page_fetcher.py`): for the supplier's site → the strain page and the
     COA page; for each site on the Sources tab → the strain page.
   - **Sweed lookup** (`sync/sweed_client.py::fetch_live_products(search_term=name)`, ~15s, real
     headless Chromium): finds the store's own listing by *name* (Sweed search also matches
     descriptions, so name matching is done in our code).
2. `app/ai/strain_generator.py` puts what was read into the prompt as labeled material and calls
   Claude **once, no tools**. `sources` and Sweed's terpene names are set by code, not the model.
3. The route returns the strain plus `research_status` (per site: used / nothing found / blocked /
   error) — shown in the UI as "What was researched"; not stored.

## Finding the right page from just a base URL
Order: sitemap.xml (also robots.txt `Sitemap:` lines; sitemap indexes followed, children ranked so
`sitemap-strains.xml` is tried first, max 6) → last path segment matches the strain's slug (≥0.92
similarity) → else common URL patterns (`/strains/<slug>`, ...). The fetched text must **contain the
strain's name** or it's reported "not found" (soft-404 / wrong-page guard). Store/menu/shop/dispensary
paths are skipped (a dispensary's price row is not a strain page). robots.txt is honoured; 403/429/503
= "blocked" and is **not bypassed**. Sitemaps are cached 6h (AllBud's is 2.8 MB / 16k strains).

## COA handling and terpene effects
No manual COA entry any more (form section removed). **Lab COA data is the source of terpene truth**: today
the supplier's COA page (e.g. Trailhead `/coa`, cut to just this strain's card so a neighbour's numbers
can't leak in); later Sweed's *full lab data* (`fullLabDataUrl` is null everywhere today — to be sourced).
Sweed's terpene *tag list* is names-only and ranks below a lab COA.
- The model copies terpene names + percentages for the **most recent completed batch** into `lab_terpenes`
  (+ `lab_batch`, saved as `strains.terpene_batch`). Code then **verifies each value**: the terpene's name
  and its percentage must sit next to each other on a fetched COA page, else it's dropped. (Limit: doesn't
  check *which batch* a pair came from; a COA laid out unlike `Name · 0.474%` just fails to verify.)
- A strain with verified lab terpenes is marked `coa_tested`.
- `terpene_effects` (second Effects line) is built ONLY from lab percentages + the full terpene reference
  guide (`app/ai/reference/terpenes_research.md`, copied whole; cached system block); code blanks it when
  there are no lab percentages. Sweed tag names alone fill `terpenes` but produce no terpene effects.
- `lineage` uses the source's own wording even when no cross is stated ("Trailhead cultivation
  selection"); null only if the material says nothing about origin.

## Cost (measured, Cherry Lady Slipper, Opus 5; $ = my estimate from token counts)
| Run | Tokens | ≈ Cost |
|---|---|---|
| Old search loop (Sonnet) | 354k in / 6.5k out | ~$1.15 |
| Standard, no guide | 4.7k in / 1.7k out | ~5–6¢ |
| Standard + guide, first run | 2.3–2.8k in + 17.7–18.2k cache-write / 1.5–1.8k out | ~14¢ (two runs) |
| Standard + guide, cache hit (within 5 min) | ~same, cache read | ~5–6¢ |
Deep-search tiers (~30¢ low, ~60–70¢ high) are planned; caps to be set from logged usage.

## Reference sites (Sources tab) — checked with the free "Check this site" button
See "Site reliability" below. Add sites at Strain Generator → Sources.

### Site reliability (my judgment from pages read 2026-09-24, not a measured score)
- **Tier 1 — primary:** the supplier's own page and lab COA (authoritative for their own strain).
- **Tier 2 — established/structured:** AllBud (big database; type, THC range, effects; lineage sparse),
  Leafwell (editorial; states the cross, terpenes, type).
- **Tier 3 — commercial/community:** cannaconnection.com (seed-shop; genetics by breeder, commercial
  bias, noisy page), cannabis.net (community/marketplace).
- **Unusable:** Leafly, hightimes.com, dutch-passion.com (bot-blocked); Weedmaps/Potguide/Strainly/Wikileaf
  (no discoverable strain page); seedfinder (good lineage DB, but URLs need the breeder — not
  findable by name alone).

## What it does NOT do
- Doesn't read JavaScript-rendered pages (text must be in the HTML), PDFs, or bot-protected sites.
- Doesn't link the Sweed match to the strain (linking needs admin confirmation, per the sync design).
- The generator is not deterministic: two runs on the same inputs differed in `lineage` and `therapeutic`.

## Key commands
```bash
journalctl -u dispo-menu-api | grep -E "strain generation|page fetch|research "   # tokens + what was read
# free dry run of a site (from /root/Dispo_menu/apps/api)
venv/bin/python -c "from app.ai.page_fetcher import check_site; print(check_site('allbud.com','Blue Dream'))"
```
Config: `STRAIN_GENERATOR_MODEL` in `.env` (default `claude-opus-5`); restart `dispo-menu-api` to apply.

## Verified 2026-09-24 (Cherry Lady Slipper, Trailhead, Opus 5; 39s, ~14¢)
Terpenes β-Caryophyllene 0.474 / α-Humulene 0.186 / β-Myrcene 0.126 from the Aug 13 batch (matches the
page), `coa_tested` true, terpene effects Relaxed/Calm/Appetite-suppressing (each traceable to the guide),
lineage "Trailhead cultivation selection". Wrinkles: "Appetite-suppressing" is a literal reading of the
guide's humulene note (odd for a menu); Reported Uses included goal-like phrases ("Inspiration/creativity").

### Wording rules tried and reverted 2026-09-24
Tried: terpene effects as menu-friendly felt sensations only (no health claims; "Munchie-blunting" instead of
"Appetite-suppressing") and Reported Uses limited to conditions/goals. It worked (~14¢ run) but was **reverted
at the user's request**: the Strain List already carries a "for informational purposes only, not medical
advice" disclaimer, so the guide's own wording (anti-inflammatory, analgesic, "appetite suppressant"...) is
acceptable. Prompt is back to the run-#4 state (terpene effects traceable to the guide via lab percentages;
Reported Uses = conditions/goals from the material). Legal review of the disclaimer/Cautions still pending.
Note the prompt cache (5-minute TTL) had expired on every run, so each run paid the ~18k-token guide write
(~14¢); back-to-back runs would be ~5-6¢.
