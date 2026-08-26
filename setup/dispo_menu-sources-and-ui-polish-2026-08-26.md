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

## Follow-up

- [ ] UI polish pass on Strain List / Strain Generator pages in progress (this session, same day)
