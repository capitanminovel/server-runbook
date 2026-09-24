# dispo_menu Strain Generator — accuracy and cost problems (2026-09-24)

## What happened
1. A single strain cost **>$1** (search/fetch loop), and switching to Sonnet 5 made results **thinner**
   (no supplier description, facts or terpenes), not just cheaper.
2. While rebuilding (our code fetches pages, one Claude call), three bugs surfaced in *testing*, before
   any user saw wrong data:
   - **COA window leaked a neighbour's numbers.** The excerpt around "Cherry Lady Slipper" on the
     supplier's `/coa` page also contained Gastronaut's and Killer Cupcake's lab values.
   - **AllBud sitemap only 159 of 16,060 URLs loaded**, so "Blue Dream" wasn't found.
   - **`terpene_effects` was generated with no terpene data** and without the reference guide attached.
3. The "Check this site" test called a cannabis.net **dispensary price row** a working strain page.

## Why
1. Server-side `web_search`/`web_fetch` re-send the growing context each round trip (~20×). The Sonnet
   run also used the fetch cap (8000 tokens/page) and model change together, so its thinness is
   unattributed (I hadn't logged which URLs it fetched).
2. - COA excerpt was a fixed ±1500-character window; cards are ~300 characters each.
   - An index sitemap's children were sorted by "contains 'strain'" and capped at 4 — AllBud has six
     such children (`strains-symptoms`, `-effects`, ...), so the real `sitemap-strains.xml` was cut.
   - The guide was only attached when terpene names were known up front, but Trailhead's COA *page*
     supplies them inside the fetched material.
3. The matcher accepted any URL ending in the strain's slug.

## Fix
- Standard mode: our code reads pages (`page_fetcher.py`), Sweed lookup by name, ONE Claude call, no
  tools; `sources` set by code. Model default back to `claude-opus-5`. Measured ~5–14¢ (was ~$1.15).
- COA excerpt now cuts at card boundaries (the line before the strain name opens each card), verified
  to return exactly one card per batch.
- Child sitemaps ranked (`strains` first, up to 6), stop at first match.
- Guide always attached in standard mode (cached); prompt forbids filling `terpene_effects` from
  reputation; `estimated_terpenes` may use names from a fetched COA/research page.
- Matcher skips `/dispensaries/`, `/menu/`, `/shop/`... paths and prefers `/strains/` URLs.

## What I learned
- Test the *positive* case: "not found" for AllBud looked correct until Blue Dream also came back empty.
- Print the exact prompt (free) before the paid call — that's how the COA leak was caught.
- Adjacent data on a page is the classic way a wrong-but-plausible value gets attributed; cut at the
  record boundary, not by distance.
- Change one variable at a time when benchmarking (model + fetch cap together = no attribution).
- Concepts: see `concepts/ssrf.md` (new fetcher's guard) and `setup/dispo_menu-strain-research-pipeline.md`.

## Follow-up
- **Live mnlegitdev south-metro repo still has the terpene-order-as-dominance bug** (RATINGS_PROMPT
  lines ~58/63/68; fixed in capitanminovel/legit-buddy-api on Aug 23) — mood ratings on the live menu
  are affected. Not touched (their repo); a proposed fix is offered but not sent.
- south-metro `misc`/lineage extras come from model memory (no search) — unverified.
- Leafly has no strain-data API (Menu/Order APIs only, paid retailer subscription) and blocks bots;
  a "paste research" box is the sanctioned alternative — not built.
- Decide whether to store per-batch COA percentages from the supplier page (top-3 only).
- Deep-search button with spend tiers (low ≈ 30¢, high ≈ 60–70¢): caps to be tuned from logged usage.
- Cautions phrase list still needs legal review; education tile still unscoped.
