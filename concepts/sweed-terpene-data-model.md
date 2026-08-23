# What Sweed's POS API Actually Exposes for Terpenes

## What it is

Sweed is the POS platform behind `shop.mnlegitcannabis.com` (and likely other
dispensary storefronts using the same white-label system). Its
`/_api/Products/GetProductList` and `/_api/Products/GetProductByVariantId`
endpoints return product data including a terpene list — but what's actually
in that list is easy to misread, and both `legit-buddy-api` and (unverified)
`dinky-dope-menu` built AI mood-rating logic on a wrong assumption about it.

## How it works

Confirmed directly against the live API (fetched real product pages, parsed
the embedded JSON) for multiple products across two brands:

- `strain.terpenes` is an array of `{id, name, filter, canonicalName}` objects.
  **That's it — no `value`, no percentage, no concentration field of any kind.**
  It's a fixed taxonomy tag list (every terpene has a static database `id`),
  not a lab-measurement result. It tells you *which* terpenes were detected,
  never *how much*.
- List **order is not concentration order**. It's arbitrary/taxonomy-driven —
  in every sample checked, items came back in ascending `id` order, unrelated
  to how much of that terpene is actually in the batch. Proven concretely: the
  same strain (Campfire Cannabis "Soap") sold as flower vs. pre-roll returned
  terpene lists in a different order purely because the pre-roll's batch COA
  happened to also detect Myrcene (id=1, sorts first) — not because Myrcene
  was more concentrated in that batch.
- Real batch-level lab data lives in a *different* place: `variant.labTests`,
  alongside `thc`/`cbd`:
  ```json
  "labTests": {
    "thc": {"value": [22.1], "unitAbbr": "%"},
    "cbd": null,
    "terpenes": {"value": [1.67], "unitAbbr": "%"},
    "tac": {"value": [21.33], "unitAbbr": "%"},
    "fullLabDataUrl": null
  }
  ```
  `labTests.terpenes` is one **aggregate total-terpene percentage** for the
  whole batch. `labTests.tac` is Total Active Cannabinoids. Both are
  populated inconsistently — present on some batches, `null` on others,
  independent of whether the strain's `terpenes` name list is populated.
- `fullLabDataUrl` (a link to the actual COA PDF, which would have real
  per-compound percentages) was `null` on every product checked. There is no
  path to real per-terpene percentages via this API today.

## How it's set up on this server

`legit-buddy-api/scraper.py`'s `_normalize_sweed_product()` reads
`labTests.thc`/`labTests.cbd` and (as of 2026-08-23) also `labTests.terpenes`
→ `total_terpenes_pct` and `labTests.tac` → `tac_pct`. A `coa_status` field
(`"confirmed"` / `"no_coa"`) is set based on whether `total_terpenes_pct` is
present — **not** on whether the terpene name list is non-empty, since the
name list can exist even when this specific batch has no lab-tested total on
file (it may be a static/legacy strain tag, not proof of testing this batch).

`docs/terpenes_research.md` in `legit-buddy-api` is a separately-maintained,
citation-backed reference (Russo 2011/2019, Kamal et al. 2018, Gertsch et al.
2008, Gadotti et al. 2021) mapping terpenes to effects — this is real research
and a legitimate source for mood-scoring rules, but it is not itself COA data
and doesn't change anything about what Sweed exposes per-batch.

## What it does NOT do

- Sweed does **not** give you per-compound terpene concentrations. Any code
  that ranks or weights terpenes by their position in the `terpenes` array is
  reading noise as signal.
- A non-empty terpene name list does **not** mean this batch has been lab
  tested for terpenes — check `labTests.terpenes` (via `total_terpenes_pct`/
  `coa_status` in this repo's schema) for that, not the name list's presence.
- Total terpene % is a single aggregate number, not a breakdown — it tells
  you "how much total terpene volume," not "which one dominates."

## Key facts to check before trusting any terpene-derived score

- Does the code path assume `terpenes[0]` is dominant? That's always wrong
  for Sweed-sourced data.
- Does a "no data" case (empty terpene list) get floored to low scores
  across the board, or handled as a distinct "not rated" / research-estimate
  case? Silently flooring makes "we don't know" look like "this is bad."
- Any other menu built on the same Sweed-scraping pattern (`dinky-dope-menu`
  is the known sibling — see
  `../incidents/2026-08-23-terpene-order-treated-as-potency.md`) should be
  checked for the same assumptions before trusting its mood ratings.
