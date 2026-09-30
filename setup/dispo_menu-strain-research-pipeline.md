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

## Cost (measured from logged tokens; Opus 5 = $5 in / $25 out per MTok, cache write 1.25x, read ~0.1x)
| Run | Tokens | Cost |
|---|---|---|
| Old search loop (Sonnet 5, $2/$10) | 354k in / 6.5k out | ~77¢ (earlier "$1.15" used wrong rates; the >$1 runs were Opus) |
| Standard, no terpene guide | 4.7k in / 1.7k out | ~6.6¢ |
| Standard + guide, **cold cache** | ~2.8k in + 18.2k cache-write / ~1.7k out | **~17¢** |
| Standard + guide, **warm cache** (previous run <5 min ago) | ~2.8k in + 18.2k cache-read / ~1.7k out | **~6.5¢** |
The terpene guide (~18k tokens) is a cached system block, so the first strain in a burst pays ~17¢ and each
further strain started within 5 minutes pays ~6.5¢ (10 strains ≈ 76¢). The cache refreshes on every read.
Gaps of 5–60 min re-pay the write; a 1-hour TTL (2x write) would only pay off with 3+ requests per hour.
Earlier estimates in this doc/commits (~14¢, ~5–6¢) used assumed rates and were slightly low.
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

## 2026-09-24 additions: product type, aliases, guide gating (measured: "MUAI WOWIE", Vapes, Tasty Gems)
- **Product type** (Flower / Pre-rolls, Vapes, Concentrates) is picked on the generate form, saved on the strain
  (`strains.product_type`), shown on the card, and filterable on the list. The Sweed lookup searches flower,
  pre-roll, vapes ("carts", id 5684) and concentrates (id 5251, lookup only — NOT added to the sync) and only
  uses a match of the chosen type; a same-name product in another type is reported "wrong type — not used".
  Supplier brand breaks ties. Sweed's category list: `Products/GetProductCategoryList`.
- **Typos and aliases:** if Sweed matches the name with a typo ("MUAI WOWIE" -> "Maui Wowie") and/or its
  description says "also known as ..." (AllBud lists it as "Maui Waui"), page lookups that missed are retried
  under those names — research sites only, so it stays a few seconds. No AI cost.
- **Terpene guide** (~18k tokens) is sent only when a supplier COA page was actually read. No COA -> no guide ->
  no cold-cache write. Measured: 12k tokens in / 636 out = **7.6¢** (3 sites read; guide skipped).
- **Vape/concentrate potency** (THC 80%+) is not shown to the model — it kept landing in Misc as if it were a
  strain fact.
- **Tasty Gems** publishes no per-strain pages (sitemap = 8 general pages) and its COAs are PDF downloads
  (`/coa’s`, punctuation now handled) — PDFs are not read. Leafly removed from Sources (blocked every run).
- Sources tab now: allbud.com, leafwell.com, cannaconnection.com, cannabis.net.
- Log now shows `est_cost` per generation and `research ... took Ns`; cost by `journalctl -u dispo-menu-api | grep "strain generation"`.

## Linking a strain to live-menu (Sweed) products (2026-09-24)
- **One explicit click per link** (docs rule: never silent auto-link). After a generation the result lists every
  live-menu product matching the strain's name and type (best first; supplier brand ranked first) with **Link**
  and **Link all**; the strain card has "Link to live menu" for older strains (~15s lookup).
- **Many per strain:** table `strain_sweed_links` (strain_id, sweed_product_id, name, brand, category).
  Flower and pre-roll are one product type but can be two products (e.g. MAC Stomper flower $85 + PR MAC Stomper
  pre-roll $18); sizes are separate ids too. A product can belong to only one strain (409 otherwise).
- **Server verifies:** the browser sends only product ids; the API accepts an id only if the live menu really
  returned it for this strain (cached 30 min per strain, else re-looked-up) and copies name/brand/category from
  Sweed, not from the request. Tenant scoping as everywhere else. At most 2 headless-Chromium lookups at once.
- `strains.sweed_product_id` stays as the "primary" link (first made; moves to the next on unlink) because the
  sync reads it; `run_sync` now also treats every link as linked (no stub re-created for a second product) and
  marks a strain nonactive only when none of its products remain. (Sync is greyed out in the UI; that path is
  code-read but not yet exercised.)
- Endpoints: `GET /api/strains/{id}/sweed-candidates`, `POST /api/strains/{id}/sweed-links`,
  `DELETE /api/strains/{id}/sweed-links/{link_id}`; list/detail responses carry `sweed_links`.

### Live-menu contents change (seen 2026-09-24)
Sweed's public storefront API (`Products/GetProductList`) returns only products that are available right now.
A Tasty Gems Maui Wowie cart matched in three runs that afternoon, then disappeared: not in any of the store's 8
categories (accessories, carts, concentrates, edibles, flower, hemp-derived THC, pre-rolls, wellness), not via a
brand search, and none of the stock-style request options tried (`stockType`, `includeOutOfStock`, ...) changed
the result. All 33 flower variants returned were "Available" with qty > 0; nothing inactive is ever returned.
(The store also listed ~143 flower/pre-roll/edible/cart products on Aug 26 vs ~43 now — a month apart, not hours.)
A sold-out product being "in the API but not active" would need a fuller Sweed API (back-office/integration key);
we have none (`SWEED_API_KEY` in .env.example is empty). Consequences: (1) a sold-out strain gets no Sweed data
(and a misspelled name gets no store-spelling/alias retry); generation still works, sparse — measured 2.9¢,
4.7k in / 217 out, nothing read; (2) links persist (the Sweed id is stored), only *finding* a match needs the
product listed; (3) link a strain while its product is on the menu; (4) idea: snapshot the Sweed listing when a
link is made so its description/terpene tags survive a sell-out.

## Sweed lab data endpoints (found by the user 2026-09-24, wired in the same day)
Both POST to `https://shop.mnlegitcannabis.com/_api/Products/<name>` with header `storeid: 434`, from inside the
real browser session (same mechanism as `GetProductList`):
- **`GetExtendedLabdata`** `{"variantId": 593666}` -> per-terpene percentages (`terpenes.values[]` with
  name/code/min/max, incl. a "Total Terpenes" row), Total THC / THCA, `fullLabDataUrl` (still null). **This is
  the "full lab data" source.** Names use ASCII prefixes ("B-Caryophyllene") -> we write them "β-Caryophyllene".
  Note it takes a **variant** id (one per size: 3.5g / 7g), not the product id our links store.
- **`GetProductByVariantId`** `{"variantId":"593666","platformOs":"web","stockType":"Default"}` -> the full product
  (description, strain flavors/terpenes/effects/scents, variants, `detailedLabDataExists`). The list call already
  carries most of this. `stockType` other than "Default" (All / OutOfStock / Inactive) returns 400, so no way found
  yet to fetch sold-out items — needs a sold-out product's variant id (the number ending its menu URL) to test.
- **Coverage today (checked per product):** per-terpene % on 14 of 40 listed products (flower 11/25, pre-rolls
  3/11, edibles 0/4). Cap Junky, Cherry Lady Slipper etc. answer with THC only. The list response does NOT carry
  `detailedLabDataExists`, so you have to ask the lab endpoint per variant.
- **Priority for lab terpenes (flower only):** (1) Sweed extended lab data — set by code straight from the
  response, no AI copying; (2) supplier COA page (model copies, code verifies name+number adjacency); (3) Sweed
  terpene tag names (presence only, no terpene effects). Saved as `terpenes` with `terpene_batch = "Sweed lab
  data"`; the terpene guide is attached whenever (1) or (2) exists. Vapes/concentrates keep tag names (a cart's
  terpene profile is the product's, not the strain's).

### Rule change (user, 2026-09-24): percentages ONLY from Sweed lab data
"COA terpenes only if in Sweed lab data since we never know what batch" — a supplier COA page lists several
batches and nothing says which one is on the shelf, so its **percentages are never used**. Sweed's extended lab
data belongs to the listed product, so:
- `terpenes` with percentages, `coa_tested`, `terpene_batch = "Sweed lab data"` and `terpene_effects` come only from
  Sweed's lab data (flower). Set by code; the model is told to return `lab_terpenes` empty and any value it sends is
  discarded. Replaces the earlier model-copies-from-COA-page path and its name+number verifier.
- The supplier COA page is still fetched, as a **reference for common terpenes**: `estimated_terpenes` = names that
  recur across its batches, else Sweed's tags, else research-page names; names only, each must appear in the
  material given (checked in code, so a name from memory is dropped). Its batch facts may still appear in Misc.
- Terpene guide (~18k tokens) is attached only when Sweed lab data exists, so most runs no longer pay the ~11¢
  cache write: a strain with no Sweed lab data costs roughly 3-6¢, one with it ~17¢ cold / ~6.5¢ warm.
- Card label: "Terpenes (lab)" when lab-measured, "Terpenes (common)" otherwise.

## Brands, visible Sources, Misc (2026-09-24, later)
- **"Supplier" is now "Brand" everywhere the admin sees it** (generate form, status rows "Brand page / Brand lab results",
  card "Brand Description"). The DB/API names (`suppliers`, `supplier_id`, `supplier_description`) are unchanged.
- **Brands list** = the cannabis brands on the live menu (flower / pre-roll / carts / concentrates). Websites saved
  after checking each with our fetcher: Trailhead, Tasty Gems, Campfire Cannabis (lakeleafcultivation.com — no sitemap,
  no strain pages), dizgo (dizgoco.com — has /strains/<name> pages), Unbound (enjoyunbound.com), Island Pezi, Lifted
  North, Lakeside Canna (lakesidecannabisco.com), Avió (aviosupply.co, given by the user). Left blank: Marawanna
  (marawanna.org is a dispensary/edibles site and broken), Redwood County Weed Co (nothing found), Northstar Grow Co.
  Edit route added: `PATCH /api/suppliers/{id}`. Hemp/wellness/accessory/edible brands were deliberately not added.
- **Sources now always show the inputs, not only pages read:** first `Brand: <name>` (linked if it has a website), then
  `Live menu (Sweed): <product> — <brand>, <category> [· lab data]`, then the pages. `SourceRef.url` is optional.
- **Misc (`interesting_facts`) must be a genuinely interesting fact** about the strain or the grow (origin/breeding story,
  awards, cultural note, where/how grown); it is no longer a place for type, effects, appearance or THC/terpene numbers.
  Null only when the material states none (grounding rule unchanged: an honest blank beats an invented fact).
  The listing's THC/CBD/total-terpene values are no longer given to the model at all.
- MAC Stomper (Sweed flower + pre-roll, lab data) read only AllBud as a *page*: Leafwell / Cannaconnection / Cannabis.net
  carry no MAC Stomper page (checked their sitemaps).

## Batch tab (2026-09-24)
Strain Generator -> **Batch (up to 4)**: rows of strain name / brand / product type, run **one at a time from the browser**
through the same `POST /api/strains/` the single form uses (no new backend). Sequential on purpose: keeps the terpene guide
in Anthropic's 5-minute prompt cache (back-to-back runs ~6.5c vs ~17c cold) and avoids piling up headless-Chromium Sweed
lookups (server cap: 2 at once). ~40s per strain; the estimate shown is 5c-8c each plus at most one ~11c cache write.
Limits: the tab must stay open (a leave-page warning shows while running; a finished strain is already saved, the rest of the
queue is lost); an out-of-credits error (503) skips the remaining rows. Upgrade path if batches grow: a server-side job
(table + worker + status route) so the page can be closed. Also possible: fetch the Sweed catalog once per batch and match
names locally (saves ~13s per strain).

## Deep search for a blank Misc (2026-09-24)
When the normal pass finds no interesting fact (`interesting_facts` null) the generator runs **one cheap web-search pass**
(`app/ai/deep_misc.py`): Haiku 4.5 with `web_search_20250305`, max 2 searches, proposes up to 2 facts about the strain's
story or grow, each with its source URL. **Nothing is kept on the model's word:** our own SSRF-guarded fetch reads each
cited page and keeps a fact only if >=70% of its content words are on that page and the page mentions the strain
(unit-tested: true fact passes; invented fact, wrong-strain page, too-short claim fail). Verified pages are added to
Sources; a row "Deep search (Misc)" shows in "What was researched" with the cost. If nothing verifies, Misc stays blank.
- **Measured (MAC Stomper):** Sonnet + 3-4 searches = 77k input tokens = **21c** (the search results are the cost) ->
  Haiku + 2 searches = 18k in / 0.3k out = **4.0c**, 15s, 2 facts verified (breeder Capulator; extraction reputation).
- Web search is billed per search (~1c, from Anthropic's list price of $10 per 1,000) on top of tokens.
- Settings: `DEEP_MISC_ENABLED` (default true), `DEEP_MISC_MODEL` (default claude-haiku-4-5) in `.env`.
- Sources order everywhere: Brand, pages read (incl. deep-search pages), then "Sweed - <name> (<size>)" last, one per
  size, each linking to the live-menu page (`.../flower-5221/<variantId>?stockType=Default`; the variant id alone works).
- Strain Sweed links now store sizes (`strain_sweed_links.variants`); the strain card's linked live menu is a dropdown,
  and the Strain List cards are collapsible (Expand all / Collapse all).

## Concentrate test findings and fixes (2026-09-24, late)
User's manual test of Dulce de Fresa (Marawanna concentrate): terpene tags only, boring Misc, 1 research page.
- **Sweed lab data exists for concentrates** (all 3 Marawanna items: Dulce De Fresa has 12 terpenes with %, total 6.464%,
  Total THC 64.4%). It had been limited to flower by an earlier decision; now used for **every product type**. Lab THC
  is still not shown to the model. A concentrate with lab data attaches the terpene guide (~11c on a cold cache).
  Sweed's own data has a typo ("Carophyllene Oxide"), normalised.
- **"1 of 4 research pages"** = 4 sites tried, 1 had a page (AllBud). Niche strains have thin coverage; the richness of
  Cherry Lady Slipper came from the BRAND's own site (Trailhead), which Marawanna doesn't have. Coverage test (free):
  Strainpedia and SeedsHereNow carry MAC Stomper + Blue Dream (added to Sources); nothing new carried Dulce de Fresa;
  Leafly/Weedmaps/most others give nothing or are blocked.
- **Boring Misc:** the model must now also return `misc_is_real_fact` (true only for a story/origin/award/cultural/grow
  detail beyond the cross, name, type, effects, flavors, appearance, numbers). If false the Misc is dropped, which
  triggers the ~4c web-search pass; if that verifies nothing, Misc stays blank.
- Research sites now: allbud, leafwell, cannaconnection, cannabis.net, strainpedia, seedsherenow.

### More research sites for niche strains (user-supplied, 2026-09-24)
For Dulce de Fresa the user found: growdiaries.com (**HTTP 403 — bot-blocked, not usable, not bypassed**),
jointcommerce.com (blog post with an "Origins and History" section — a good Misc source, but a marketing blog, so
lower reliability; the grounding rules still require facts a page states directly), kalikori.me (a grower's page,
541 chars: genetics Dulce de Uva x Strawberry Guava, "retired"). Blog titles don't match our page-name rule, so
matching now also accepts `<strain>-(strain|cannabis|weed|marijuana|cultivar|review|guide|info)...` page names
(tested: `blue-dream-haze-strain-...` is NOT taken for Blue Dream). Both sites added to Sources; Dulce de Fresa now
reads AllBud + JointCommerce + Kalikori (3 pages). Research sites: allbud, leafwell, cannaconnection, cannabis.net,
strainpedia, seedsherenow, jointcommerce, kalikori.

### Common terpenes from CannMenus (user, 2026-09-25) — names only, only without lab data
cannmenus.com/strains/<name> publishes a strain's terpene profile as "the statistical mean" of lab COAs for products
containing it (coverage: 7 of 8 test strains). `app/ai/common_terpenes.py` parses its "Full Terpene Distribution" block
in code (top 8 by average). **When there is no Sweed lab data, those terpene NAMES fill the Terpenes line ("(common)");
the percentages are deliberately NOT shown or stored** — an average across products is not this product's number
(e.g. MAC Stomper: average beta-caryophyllene 1.15% vs 0.449% in its own Sweed lab data). With Sweed lab data, the real
lab numbers are used and this is ignored. No extra AI call: our code fetches and parses the page, so the only cost is the
page text in the prompt (~2k chars, roughly half a cent). Terpene-based effects still need real lab percentages.

## Status model, archive/delete, test harness (2026-09-25)
- **"Draft" is retired.** Generated profiles are final outputs an admin can edit. `strains.status` is now `active` =
  linked to at least one live-menu product, `nonactive` = not linked (and, once the sync runs, linked-but-sold-out).
  Linking flips it to active, unlinking the last link flips it back. Existing draft rows were migrated. (The enum value
  `draft` remains in Postgres — it can't be dropped — and is unused.)
- **Archive / delete:** `strains.archived_at` (null = live). Archive keeps everything and hides it from the main list;
  the Archived filter shows them and offers Restore. **Permanent delete needs two deliberate steps in the UI ("Step 1 of
  2" then typing the strain's name) and the API enforces both rules again:** the strain must already be archived and
  `confirm_name` must match (`POST /api/strains/{id}/delete-permanently`). Links cascade; flags/log rows are removed.
- List filters: All / Active / Not active / Archived (+ product type). `run_sync` no longer creates drafts.
- **Free test harness:** `apps/api/scripts/research_check.py` runs 10 named cases through the research layer with no AI
  cost (about 6 minutes: one headless browser lookup each). Cases and purpose: `docs/strain-generator-test-cases.md`.
  Result on 2026-09-25: 10 pass, 0 warn, 0 fail.

## Incident: strain list silently failed to load under quick clicking (found by the new UI test, 2026-09-25)
**What happened:** the saved browser test failed at random points: the list showed no strains, the delete "did nothing".
**Why:** nginx's rate limit for the paid generate call (`dispo_ai_generation`, 30/min + burst 10) was attached to the
exact path `/api/strains/`, which is ALSO the list request (GET). The comment claimed normal use "never notices it",
but Archive / Restore / filters / link each refetch the list, so ~20 quick clicks tripped it. nginx answered 503 with no
CORS headers, which the browser reports as a bare "Failed to fetch" -- and the app never saw the request.
**Fix:** `map $request_method` gives every non-POST an empty key (nginx does not limit an empty key), so only POST
(generate) is limited; a new location covers `POST /api/strains/<id>/redo` (it calls the AI too and had no limit). Proven:
60 rapid list GETs all 200; 45 rapid POSTs -> 13 reached the app, 32 got 503.
**Gotcha learned:** nginx refuses to reload when an existing limit zone changes its key type -- `systemctl reload` still
reports success and the old workers keep the old config (the reload error is only in /var/log/nginx/error.log:
`limit_req ... uses the ... key while previously it used the ...`). Fix = a new zone name (`dispo_generate`).
Always check `ps -o lstart` of a worker / the error log after a reload.
**What I learned:** the browser test earned its keep in the first hour; also the list must refresh silently after a card
action (a "Loading…" flash collapsed every card), and a badge must follow the server-side status change.

## Status model, edit, refresh — correction to the earlier note (2026-09-25)
The earlier section mentioned `run_sync` and flags: both are now removed (see `setup/dispo_menu-roadmap.md`, "Done 2026-09-25").

## Same strain, other product (2026-09-27)
**Why:** Octane Mintz is Trailhead's MN-only cultivar. As Trailhead flower the research read trailheadmn.com's strain page →
rich profile. As Marawanna rosin (Marawanna has no website) research only had the Sweed listing → no flavors, aroma or
fact. Research used to read only the SELECTED brand's site, and general strain sites don't know MN-only cultivars.

**Now (all free, no AI):**
1. `earlier_profiles`: if we already profiled the same strain name (other brand/type), its source pages are re-read
   (never lab/COA pages — those are another product's batches) and its strain-level facts are shown to the AI, marked
   "strain facts carry over; lab numbers, format and maker do NOT".
2. `other_brand_websites`: when the chosen brand's own site gives nothing, the dispensary's other brands' sites are
   searched for the strain page (role `other_brand`, shown as "Another brand's page (likely the grower)"). Only hits
   are listed, plus one summary row "Other brands' sites · N checked".
3. **Fill empty fields** button (Strain List, admin, in an opened card): copies lineage, type, flavors, aroma, effects
   and Misc fact from our other profile of the same strain into EMPTY fields only. Never overwrites; lab terpenes and
   the maker's description stay per product. Adds a source line saying where the facts came from.

Code: `app/ai/strain_siblings.py`, `research.gather_research(...)`, `page_fetcher.read_known_page / find_on_other_brand_sites`.
Tests: `research_check.py` case "Octane Mintz — Marawanna / concentrates" (must find trailheadmn.com);
`same_strain_check.py` (8 checks on two temporary test strains).

## Lab reports (COA PDFs) on strain cards — added 2026-09-27, dev only
- **What:** an admin uploads a COA PDF on a strain card; the server pulls its text out once (pypdf, free) and the
  "Redo profile using lab report" button (one paid call, ~10-17c, asks first) gives it to the Generator as research.
- **Never the menu's terpene %:** those still come only from Sweed lab data (we can't be sure the COA is the batch on
  the shelf). The prompt says so, and code sets lab terpenes anyway.
- **Scanned COAs** (a picture, no text layer) are stored but marked "scanned" and not used. Reading them would need OCR
  (too heavy for this droplet) or sending the PDF to Claude (~2-6c/generation), which we don't do without asking.
- **Safety:** same upload rules as Education/Schedule (content-checked PDF, 25 MB, quota, random names). Parsing runs
  in a separate process with a 20 s timeout and a 512 MB memory cap, so a hostile PDF can't hang the API.
- **Switch:** `FEATURE_COA_FILES=true` in `/opt/dispo-menu/api/.env` (dev). Off by default; off = routes 404, research ignores it.
- **Test:** `apps/api/scripts/coa_check.py` (free; temp strain, 11 checks).

## Other names, Pages to read, lineage check, Find other names — added 2026-09-27 (dev)
- **Why:** the store sells "Double Sour Grape"; research sites only know the same plant (Grape Crinkle x Sour Stomper)
  as "Double Grape". Found by the team testing on the demo.
- **Also known as** (`strains.aka`, admin-entered): research retries every site that found nothing under each other
  name. **Lineage check** (`app/ai/lineage_check.py`): a page found under another name is used only if it names OUR
  parents (from the strain's lineage, else the store description's cross). Different parents -> "not used"; no parents
  stated -> "can't confirm, not used". Free.
- **Pages to read** (`strains.reference_urls`): links an admin found by hand; same safe fetcher (http(s), no private
  addresses, robots.txt); used only if the page names the strain, an other name, or its parents. Free.
- The name box now refuses two names ("A, B") and points to Also known as.
- **Find other names** (paid, `FEATURE_NAME_SEARCH`, dev only): Haiku + Anthropic web search (max 2 searches) proposes
  pages for the same parents; our code reads each and keeps only names whose page names our parents. Nothing is saved
  until the admin clicks Use. **Measured: $0.039, 13 s** for Double Sour Grape -> found Double Grape (Strainpedia);
  Leafly/GrowDiaries unreadable (bot blocks, respected). ~2c of that is the two searches -- one search would be ~2c.
- **Redo profile** button on every strain card (asks first, ~10-17c): research + AI again using other names, pages and
  COAs; hand-edited fields are kept.
- Before turning Find other names on for the demo: add an nginx rate limit for `/api/strains/<id>/find-other-names`
  (it's admin-only and behind dev basic auth today) and decide whether it counts toward the 10-generation cap.
- **Automatic (same day):** when research is thin (<2 pages) and the parents are known, the name search runs BEFORE
  the AI writes; confirmed names are researched (free) and saved as Also known as. Tested on a fresh Double Sour Grape
  with nothing typed: 0 -> 2 pages, full profile (flavors, effects, a real Misc fact), ~14c total, 79 s. The Generator
  result now has a "Thin research" box (another name / a page -> Save & redo).
- **Costs are developer-only:** `SHOW_COSTS=true` in dev shows each run's cost (result screen, Redo message) and the
  cent figures in prompts; off (default) on the demo/production, where the server doesn't even send them.
- **Compact terpene guide tested, NOT adopted (2026-09-28):** `terpenes_compact.md` (57% smaller, ~6c cheaper on a
  cold lab-data run). Side by side on the same input (Soap, Fight Club; 31c): core effects matched, but the compact one
  added "Appetite-suppressing" for Soap (0.36% humulene, animal-study evidence, and cannabis usually raises appetite).
  Accuracy over ~6c, so the full guide stays. The compact file is kept for a later retest (e.g. with a prompt rule
  limiting terpene effects to the dominant terpenes).
- **Real costs seen in dev:** thin strain with auto name search ~14c (Double Sour Grape) to ~25c (Tropical Fresa, cold
  guide + long output); name search alone ~3.9c; write-up 3.5-21c depending on pages, lab data and cache.

## All-in-one research for hard-to-find strains (2026-09-30, dev)
- **Why:** team test on the demo — "Grape Canyon Zkittlez" came out thin; Find other names later found "Grape
  Zkittlez"; the admin then RENAMED the strain to steer the research, which lost the live-menu match. Same with
  "Gorilla Zkittles" (typo of the menu's "Gorilla Zkittlez", which the lookup didn't find at all).
- **Menu spelling check:** Sweed's search is whole-word ("Zkittles" misses "Zkittlez"). When the typed name finds
  nothing, `lookup_sweed` compares it with every live-menu product name (one direct HTTPS read, 1–4 s). >= 0.88
  similar -> that product is used (typed name kept, menu spelling saved as Also known as); 0.72–0.88 -> "did you mean"
  shown under the result (rename + redo in one click).
- **Use & redo:** one click saves the other name + its page and redoes the profile; the strain's name never changes.
- **Name search** always tries two phrasings: `"Grape Ape" "Zkittlez" strain` and `Grape Ape x Zkittlez cannabis strain`.
- **Tested (dev, 24c):** one Generate of "Grape Canyon Zkittlez" -> menu match, Grape Zkittlez found and researched
  before writing, full profile. Test profile deleted.
