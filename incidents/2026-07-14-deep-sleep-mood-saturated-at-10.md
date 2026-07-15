# Deep Sleep Mood Showed 10 for Nearly Every Strain (South Metro Menu)

**Date:** 2026-07-14

## What happened

Right after adding the "🌙 Deep Sleep" mood filter to `mnlegitdev/legit-cannabis-south-metro-menu` (porting it from `dinky-dope-menu`), almost every product showed a rating badge of **10** when filtering by Deep Sleep — an obviously saturated, undifferentiated result.

## Why

The frontend picks a strain's mood score two ways (`build_preview.py`, `applyFilters()`):
```js
const score10 = claudeRating != null
  ? claudeRating
  : Math.min(10, Math.round(moodScore(card, mood) * 1.5));
```
`claudeRating` comes from `docs/strains_enriched.json`'s `mood_ratings` — a real, calibrated score generated once by Claude when the strain was first enriched. If that's missing, it falls back to a formula (`moodScore()`): position-weighted terpene match + a flat per-effect bonus, internally already capped at 10, then **multiplied by another 1.5x** before a second cap.

`deep_sleep` was added to the enrichment prompt (`enrich_strains.py`) the same day it was added to the UI — but `enrich_strains.py` only rates a strain **once**, ever (`"mood_ratings" not in existing[k]` — an all-or-nothing check at the dict level, not per-mood-key). Every one of south metro's 31 currently-stocked strains (109 historical entries total) was enriched *before* `deep_sleep` existed, so **100% of cards** were hitting the formula fallback — and any Myrcene-forward strain (a large fraction of any menu) saturates that formula almost automatically. Confirmed by comparing against `dinky-dope-menu`, which has had `deep_sleep` longer: strains rated by Claude *after* it was added there show a normal 2–8 spread, never maxed.

## Fix

Wrote `backfill_deep_sleep.py` (in the `legit-cannabis-south-metro-menu` repo) — a one-off script that reuses `enrich_strains.py`'s existing `rate_moods()` call and merges **only** the `deep_sleep` field into each strain's existing `mood_ratings`, leaving every other already-generated field untouched. Ran it once against all 109 historical strain entries using the `ANTHROPIC_API_KEY` from `/opt/legit-buddy-private/.env` (reused with the owner's OK — flagged first since it belongs to a different app). Result: realistic 2–8 spread, no more saturation. Rebuilt and pushed `docs/index.html`.

## What I learned

- A fallback formula tuned to *feel* comparable to a calibrated AI rating (via an extra ×1.5 multiplier) can badly overshoot when it's the *only* signal in play, not a rare fallback.
- "Enrich once per strain, never re-run" (a deliberate design to avoid overwriting AI content / re-spending API budget) has a blind spot: any newly added mood/field has zero coverage on every pre-existing strain until specifically backfilled.

## Follow-up

**This will recur for any future new mood added to this pipeline** — on this repo, `dinky-dope-menu`, or any menu built on the same `build_preview.py` / `enrich_strains.py` pattern. When adding a new mood going forward:
1. Add it to `build_preview.py` (chip, `MOOD_MAP`, `MOOD_INFO`) and `enrich_strains.py` (prompt + schema) as usual.
2. Also run a `backfill_<mood>.py`-style script (copy the pattern in `backfill_deep_sleep.py`) against existing strains immediately — don't rely on the normal pipeline to catch up, it won't.

`dinky-dope-menu` was not audited for this same issue on any of its *other* moods — only confirmed `deep_sleep` there already had partial real coverage. Worth spot-checking if a new mood is ever added there too.
