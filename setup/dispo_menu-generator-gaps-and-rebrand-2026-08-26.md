# dispo_menu — Strain Generator Gaps Closed + Dashboard Rebrand

**Date:** 2026-08-26

## Strain Generator gap audit

User asked directly: did we hit everything planned for the Strain Generator? Checked the actual
code against `docs/strain-list-tile-spec.md` line by line rather than relying on memory. Found
two real gaps (not just polish) and two known-stub items:

1. **Grower field** — spec lists "Grower (if different from supplier)" as a manual-entry field.
   The backend (`app/routes/strains.py`) already supported `grower_id` end-to-end — tenant
   validation, passed into the AI prompt as a grounding hint, stored on the strain — but the
   frontend form (`GenerateStrainPage.tsx`) never had an input for it. Fixed: added a dropdown
   defaulting to "Same as supplier."
2. **`misc` field** — spec: "admin free-text, not AI-generated." The column existed since the
   very first migration (2026-08-02) but nothing ever wrote to it — not the form, not the insert
   statement. Also sat unresolved on the docs' own "open items" list the whole time. Fixed: added
   as a plain text input, stored as-is, never touched by the AI call — resolves that open item too.
3. **Edit** (`PUT /api/strains/{id}`) — still a stub, not addressed this round (wasn't part of what
   was actually asked for — login/generator/list — flagged rather than silently left implied-done).
4. **Redo** (`POST /api/strains/{id}/redo`) — still a stub, same reasoning.

## Dashboard rebrand for the dispensary demo

- Investigated legit-buddy-api for an actual logo image file to reuse — **none exists**. Their
  "logo" (`docs/index.html` in `/opt/legit-buddy-private`) is pure CSS: a leaf emoji (🍃) in a
  teardrop-shaped colored badge, brand color `#1a7a4a` (light) / `#4ade80` (dark). Recreated that
  exact treatment in dispo_menu's dashboard rather than fabricating an image asset that doesn't
  exist — told the user this directly rather than silently substituting something else.
- Header: "Legit Cannabis Demo" + leaf badge, left-aligned. `dispo_menu` moved to a small footer
  credit line at the bottom of the dashboard (previously the page's main `<h1>`).
- Page background changed from plain white to a soft sage tint (`#eff8f2` light / `#0f1a14` dark,
  set via the existing `--bg` CSS variable in `index.css` — applies app-wide, not just the
  dashboard, so login/strains pages get it too for visual consistency).
- Tiles got a white card background + subtle shadow + brand-green hover state, instead of a plain
  bordered box on the same background as the page.
- User email + logout button compacted into a small top-right corner (was full-size text + button).

## Follow-up

- [ ] Edit and Redo strain endpoints are still stubs
- [ ] Cautions phrase list still needs legal/compliance review (flagged repeatedly, still open)
- [ ] Only 2 of 6 dashboard tiles have custom icons (Strain List, Strain Generator)
