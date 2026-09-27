# dispo_menu: research read wrong-strain pages, and missed JointCommerce pages (2026-09-27)

Found while profiling all 27 unprofiled live-menu products in a Claude Code session (no API spend).

## What happened
1. **Wrong pages accepted.** When a guessed page address (e.g. `strainpedia.com/guava/`) redirected somewhere else
   (`/guava-berry/`, a seed-shop page, a sign-in page), research used whatever page it landed on. Old profiles
   that had saved those bad pages as sources pulled them back in on re-research.
2. **Pages missed.** JointCommerce's sitemap is split into ~40 files. The finder only opens 6 of them, and ranked
   `sitemap-strain-0.xml` low because it expected a file named exactly `strain`. Burger Breath's guide lives in
   `sitemap-post-9.xml`, so it was never seen. Its URL (`burger-breath-by-atlas-seed-a-comprehensive-strain-guide`)
   also wouldn't match because "by" wasn't an allowed word after the strain name.

## Why
- Redirects: `find_strain_page` trusted the final URL without checking it was still this strain's page.
- Sitemaps: `_child_rank` compared the file name before stripping the `-0`/`-9` page number, and
  `MAX_CHILD_SITEMAPS = 6` was too small for big split sitemaps.

## Fix (`apps/api/app/ai/page_fetcher.py`, both the API Generator and session runs use it)
- `_still_this_strain(url, slug)`: after a redirect, and before re-reading an earlier profile's saved source, the
  URL must still be this strain's page, otherwise it's skipped with a note ("redirected to a different page").
- `_child_rank` strips trailing page numbers (`strain-0` -> `strain`), `MAX_CHILD_SITEMAPS = 16`, and `by` added to
  the allowed follow-on words (`<slug>-by-<breeder>-...`). A cold JointCommerce search now takes ~19 s once, then is
  cached 6 h.
- Guava #159 (bad sources) deleted and re-saved as #170.

## What I learned
- JointCommerce "comprehensive strain guides" are often AI-written filler: Burger Breath's said it smells like
  "grilled meat" and quoted invented survey stats; Wilfunk's admitted it was "extrapolated". Treat JointCommerce as
  the weakest source, and use the store listing when it conflicts.
- Some strains genuinely have no web presence (Sunset Tea, Swedish OG, Double Sour Grape). The honest profile is a
  thin one, and no fixed finder changes that.

## Follow-up
- Show admins a "thin research" warning, and let them add a page for the research to read (proposed).
- Lineage-based page check: reject a research page whose stated parents contradict the store's lineage.
- 27 new strains are saved as `nonactive`; linking them to live-menu products is still an admin click.
