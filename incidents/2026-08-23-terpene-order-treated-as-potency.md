# Mood Ratings Silently Biased by Fake Terpene "Dominance" Order

**Date:** 2026-08-23
**Repo:** `capitanminovel/legit-buddy-api`

## What happened

User asked me to check how terpene percentages were being pulled now that
COAs are adding them, using "Soap" and a new product ("Frostberry Fuel") as
examples. Investigation turned up two separate, real bugs — not just a
missing-data gap:

1. `scraper.py` was reading `labTests.thc`/`labTests.cbd` from the Sweed API
   but silently discarding `labTests.terpenes` (total terpene %) and
   `labTests.tac` (Total Active Cannabinoids %) — both present in the same
   response, never captured.
2. Both `enrich_strains.py`'s AI mood-rating prompt (`RATINGS_PROMPT`) and a
   client-side JS fallback formula in `build_preview.py`
   (`terpenePositionScore`) treated the *position* of a terpene in Sweed's
   `terpenes` array as a concentration/dominance signal ("terpene listed
   first is most concentrated"). Proven false: the same physical strain
   (Campfire Cannabis "Soap") sold as flower vs. pre-roll got *different*
   mood scores purely because the pre-roll's COA happened to also detect
   Myrcene, which has a lower taxonomy ID and sorts first — not because it
   was actually more concentrated. See
   `../concepts/sweed-terpene-data-model.md` for the full mechanism.

This directly relates to the prior `2026-07-14-deep-sleep-mood-saturated-at-10.md`
incident — same `build_preview.py` fallback-formula code path, different bug.

## Why

Whoever wrote the original `RATINGS_PROMPT`/`terpenePositionScore` logic
assumed Sweed reports terpenes sorted by lab-measured concentration (a
reasonable assumption for some COA formats) without verifying it against
this specific API. It doesn't — the array is presence-only, ordered by an
internal taxonomy ID. Also found: the rubric itself didn't reflect the
citation-backed research already sitting in `docs/terpenes_research.md` —
missing trans-Nerolidol as the strongest documented anxiolytic correlate
(Kamal et al. 2018), no penalty for Guaiol's documented negative correlation
with anxiety relief, and missing Bisabolol/Camphene for pain (Gadotti et al.
2021) despite the same doc citing them for that mechanism.

## Fix

- `scraper.py`: capture `total_terpenes_pct`, `tac_pct`, and a `coa_status`
  (`confirmed`/`no_coa`) field based on whether *this batch* has a real
  lab-tested terpene total — not on whether the terpene name list is
  populated (that list can exist without a matching COA on file).
- `enrich_strains.py`: rewrote `RATINGS_PROMPT` to score by terpene
  *presence* only, explicitly telling the model the list is unordered.
  Corrected the rubric itself against `terpenes_research.md` (added
  Nerolidol/Guaiol/Bisabolol/Camphene per the citations above). Added
  `RESEARCH_RATINGS_PROMPT` — for products with zero COA terpene data
  (some edibles), instead of flooring every mood to its "absent" score,
  ask Claude to estimate from general strain-genetics knowledge
  (lineage/breeder/seed-bank info), conservatively, clearly marked via
  `coa_status=no_coa` on the live menu. Added per-terpene-signature caching
  so products sharing an identical COA profile aren't rated twice (cut a
  full re-rate from ~154 naive API calls to 78 actual calls).
- `build_preview.py`: fixed `terpenePositionScore` → presence-based
  `terpenePresenceScore` (this is only a fallback for a strain Claude hasn't
  rated yet, per the pattern from the deep-sleep incident). Corrected the
  staff-guide "How Mood Scores Are Calculated" text, which was explicitly
  teaching the false position-weighted claim to staff. Added a "COA
  Confirmed {pct}%" / "No COA" badge to product cards and the strain detail
  modal.
- Did a full mood-rating redo: pushed the fix, then `gh workflow run
  daily-scrape.yml` to trigger it immediately rather than waiting for the
  next cron tick. Live re-scrape picked up real `total_terpenes_pct`/`tac_pct`
  for current inventory; enrichment step made 78 real API calls (76 reused
  via the new cache) to re-rate all 154 strains under the corrected rubric.

## What I learned

- Never trust that an array from a third-party API is sorted the way its
  consumer assumes — verify against the raw response, not the UI that
  renders it.
- A citation-backed research doc sitting in the repo (`terpenes_research.md`)
  is only as good as whether the actual scoring code was cross-checked
  against it — it wasn't, until asked to explain the rating methodology.
- Per-terpene-signature caching turns "redo all ratings" from an
  expensive, dreaded operation into a cheap one — worth building in from
  the start on any similar enrichment pipeline.

## Follow-up

- **`dinky-dope-menu` was flagged as sharing this exact `build_preview.py`/
  `enrich_strains.py` pattern in the prior deep-sleep incident and was never
  audited for it.** Worth checking whether it has the same
  `terpenePositionScore`-style position-weighting bug and the same
  `labTests.terpenes`/`tac` gap.
- `fullLabDataUrl` (real COA PDF link) was `null` on every product checked —
  if per-compound terpene percentages are ever needed, there's currently no
  path to them via this API; would require sourcing COA PDFs some other way.
- As more batches get lab-tested terpene totals over time, the "No COA"
  badge should organically flip to "confirmed" — worth spot-checking in a
  few weeks that the ratio is trending the right way.
