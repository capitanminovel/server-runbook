# dispo_menu — Real Sources on Generated Profiles + UI Polish

**Date:** 2026-08-26

## Sources feature

User asked for references on generated strain data, specifically wanting the supplier/grower's
own site checked when relevant (gave a real example: trailheadmn.com/strains/cherry-lady-slipper).

Added `web_search` (server-side tool) to the Strain Generator call, combined with structured
output in one request per Anthropic's documented "tools + output_config together" pattern.
Switched from `client.messages.parse()` to raw `client.messages.create()` + a manually-passed
JSON schema, since server tools aren't a documented `.parse()` combination — hit one schema gap
doing this: `output_config.format` requires `additionalProperties: false` explicitly on every
object, which `.parse()` patches automatically but raw schema passing doesn't. Added a small
recursive fixup for it.

New `sources` field: real `{title, url}` pairs the model actually got back from search this call
— explicitly never a citation recalled from training data. Prompt biases search toward the
supplier/grower's own website first (`suppliers.website` already existed in the schema, unused
until now) since that's the authoritative source for their own product.

**Verified against two real cases:**
- Blue Dream (well-documented) → correct lineage (Blueberry x Haze) + real citations to
  Leafly/Weedmaps/AllBud/Strainpedia.
- Trailhead's actual "Cherry Lady Slipper" (the user's own example) → searched multiple times
  (confirmed via raw response inspection — `server_tool_use`/`web_search_tool_result` blocks
  showed real queries and real, non-matching results), correctly returned empty sources and null
  lineage. Fetched the real product page directly to confirm: it genuinely has no lineage/terpene
  data published, just "refer to batch label" boilerplate — the empty result was correct, not a
  bug.

New `strains.sources` JSONB column, migration `6b12e34164bc`.

**This first version had a real bug the user caught by re-testing the exact same case.** Fetched
the actual Trailhead page myself earlier to confirm it had no real data — true — but never
verified the *reverse*: that the model would notice and use it when it *did* exist. Regenerating
Cherry Lady Slipper again came back with no sources and a description that didn't match the real
page at all. Root cause: `web_search` only returns short result snippets, not full page content —
the model found the right URL but never actually read it, so specific real facts (batch number,
grow location, harvest details) never reached it. Fixed by adding `web_fetch` (max_uses=3)
alongside `web_search`, with an explicit prompt instruction to fetch a promising URL before
writing the description rather than trusting the snippet. Re-verified against the same exact
case: description now correctly says "grown indoors in Brainerd, Minnesota... Batch 001
harvest... expression varies harvest to harvest" (matches the real page), and `sources` correctly
cites the real URL.

**Lesson:** verifying the empty-case (no data exists, so nothing should be cited) is not the same
as verifying the positive case (data exists, and it actually gets used). Both need a real test
before calling a grounding feature done.

## Strain List crash (same day, separate incident)

User reported the Strain List "loads then goes away." Root cause: `StrainCard` called
`strain.sources.length` unconditionally; 28 of 30 real strains in the dev database predated the
`sources` column and had `sources: null`, not `[]`. An uncaught TypeError on every card in the
list crashed and unmounted the entire page — not an auth or network issue, a JS render crash from
a schema-migration gap (new nullable column, old rows never backfilled).

Fixed three ways at once: null-guard in the component, `sources` made `NOT NULL DEFAULT '[]'`
at the DB level (migration `f475572c2eac`), and the one insert path that had never set it
(`app/sync/run_sync.py`'s stub-strain creation) fixed so this can't recur for future rows.
Backfilled the 28 existing null rows.

## Follow-up

- [x] UI polish pass on Strain List / Strain Generator pages — done, same day
- [ ] Should audit other JSONB array columns (flavors, aroma, effects, therapeutic, cautions) for
      the same nullable-without-backfill risk if the schema changes again in the future — none
      are currently null (verified), but nothing stops a future migration from repeating this
