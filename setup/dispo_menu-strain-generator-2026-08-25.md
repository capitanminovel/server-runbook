# dispo_menu — Strain Generator + Real Strain List

**Date:** 2026-08-25

## What this is

Built the Strain Generator (the one-call AI strain profile generation the docs specced) and
wired the Strain List tile to real data, for a dispensary demo. Session also included auditing
a sibling project's AI-generation code for lessons before writing dispo_menu's own prompt.

## API key: shared with legit-buddy-private

dispo_menu's `CLAUDE_API_KEY` is the same key already in `/opt/legit-buddy-private/.env`
(`ANTHROPIC_API_KEY`) — same Anthropic account, reused rather than provisioning a second key.
**Both services' usage now shows up on one account** — worth remembering if usage/billing ever
needs to be attributed per-service.

## Research: audited legit-buddy-api's enrich_strains.py before writing the prompt

Found via a past Claude Code session transcript (not a repo file) that legit-buddy-api fixed a
real bug on 2026-08-23: its `RATINGS_PROMPT` told Claude that Sweed's terpene list order encodes
concentration/dominance ("terpene listed first is most concentrated") — false. Sweed's API only
reports presence/absence, unordered. The mood-scoring prompt was quietly acting on a made-up
signal until that session caught it.

Applied directly to dispo_menu's Strain Generator prompt (`app/ai/strain_generator.py`):
- Explicit instruction that COA terpene lists are unordered/presence-only.
- A "don't guess" grounding instruction on **every** generated field, not just when there's no
  COA data — legit-buddy-api only had this instruction on its no-COA path
  (`RESEARCH_RATINGS_PROMPT`), which is an asymmetry, not a deliberate choice; its COA-tested path
  (`RATINGS_PROMPT` / `PROFILE_PROMPT`) has no equivalent guard and can still hallucinate lineage
  even with real lab data present. Verified this actually works: a nonsense strain name with no
  COA data correctly came back with `lineage: null` and empty effects/flavors/etc. rather than
  inventing anything, with an honest description explaining why.
- Kept dispo_menu's existing one-call design (already decided, not something this research
  changed) — legit-buddy-api splits profile generation and mood-rating into two separate calls,
  which the docs already flagged as worse (two chances for the model to contradict itself).
- Cautions field: legit-buddy-api's equivalent (`"negative"`) is still fully freeform AI text,
  confirming dispo_menu's fixed-phrase-list approach is the right call, not something to relax.

## Cautions: fixed phrase list, enforced structurally

`app/ai/cautions.py` — a Python `Enum` of ~12 caution phrases. The Strain Generator's Pydantic
output model types `cautions: list[CautionPhrase]`, so the **JSON schema itself** constrains the
model to that closed set — not just a prompt instruction asking nicely. This list is a first
draft, not legal/compliance-reviewed; flagged to the user for review.

## Structured output approach

Used `client.messages.parse()` with a Pydantic model (`GeneratedStrainProfile`) as `output_format`
— the SDK validates the response against the schema automatically. This is a cleaner approach
than legit-buddy-api's, which asks for "ONLY valid JSON" in the prompt and parses by hand with a
regex that strips markdown code fences, with no post-parse validation on the string fields at all.

## Schema gap found and fixed: suppliers had no classification field

`docs/strain-list-tile-spec.md` says `strains.prevalence_type` is "derived from supplier
classification, not generated" — but `suppliers` never had a classification column to derive it
from. Added `suppliers.classification` (same `indica`/`sativa`/`hybrid` enum now shared with
`strains.prevalence_type`, which was previously plain text) via migration `15faef807f8a`.

## Effects vs. Therapeutic — resolved

Another docs open item. Landed on: **Effects** = felt/psychoactive sensations (short adjectives —
Relaxed, Uplifted, Sleepy). **Therapeutic** = medical/wellness framing (conditions/goals — Chronic
pain, Insomnia, Anxiety relief). Both are genuinely AI-generated in dispo_menu (unlike
legit-buddy-api, which scrapes raw Sweed tags for something effects-like and only generates
`therapeutic`) — dispo_menu has no live Sweed sync yet to lean on for that at strain-creation
time, since "Generate Strain Profile" happens before a Sweed link exists.

## Tenant isolation reminder, applied in code

Every route that accepts a foreign key from the client (`supplier_id`, `grower_id`) re-checks
`dispensary_id` before using it (`_get_tenant_supplier` in `app/routes/strains.py`) — in the
current shared-database model, that check is the *only* thing stopping one tenant from reading
another tenant's row by guessing an id. Verified: passing a nonexistent/foreign `supplier_id`
correctly 404s rather than silently succeeding.

## Verified end-to-end (live deployment, not just locally)

Real generation calls against real strains during testing — Blue Dream (lineage: Blueberry x
Haze, correct), Wedding Cake (lineage: Triangle Kush x Animal Mints, correct) — both via the
actual API, with `prevalence_type` correctly coming from the supplier record rather than being
generated. One test strain (Wedding Cake, supplier "Northstar Grow Co") now exists in the shared
dev database as a result of this testing — left in place as a working demo example rather than
cleaned up.

## Follow-up (added later same session)

**Real COA terpene percentages, not just names.** The user asked directly: does this handle
"full lab data" better than legit-buddy-api could? Answer at the time was no — dispo_menu's
`terpenes` field mirrored legit-buddy-api's names-only limitation. But that limitation exists
there because it's an automated Sweed scraper stuck with whatever Sweed's API returns
(presence/absence only). dispo_menu's Strain Generator is a *manual* admin entry form — when an
admin has a real COA in hand, there's no reason not to capture the actual lab percentage per
terpene. Changed `terpenes` to always store `[{name, percentage}]`, percentage null only when
it's an AI estimate (no COA available) rather than a real lab value — one uniform shape
regardless of source, so nothing downstream needs to branch on `coa_tested` just to read this
field. The generation prompt now explicitly weights per-terpene percentage and total terpene %
as real signal. Verified live: a real strain (Gelato) generated with real percentages produced a
description that explicitly referenced the actual lab values ("caryophyllene-forward," "1.9%
total terpene load") — confirming the model is genuinely using the data, not just ignoring it.

**Dashboard tile icon.** Added a user-provided generated image (cannabis leaf + gears, teal-to-
purple gradient) as the Strain List / Generator tile's icon. Resized from 1024x1024 down to
128x128 with Pillow (installed in a throwaway venv just for this, not added to the app's own
requirements) before adding to `apps/admin/src/assets/`.

## Follow-up

- [ ] Cautions phrase list needs actual legal/compliance review, not just Claude's first draft
- [ ] `PUT /api/strains/{id}` (edit) and `POST /api/strains/{id}/redo` are still stubs
- [ ] No supplier management UI beyond inline creation in the Strain Generator form
- [ ] Sync-from-Sweed still not built — "Generate Strain Profile" is the only strain-creation path right now
- [ ] Only the Strain List / Generator tile has an icon — the other 4 dashboard tiles don't yet
