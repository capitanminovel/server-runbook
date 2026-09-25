# dispo_menu — roadmap and open items (kept current)

## Needed, decided to do LATER
- **Onboarding plan for new dispensaries / new products** (decided 2026-09-25: "do later, just mark as needed").
  Question to settle: how to profile a whole menu (100-200 products) cheaply. Options noted, none chosen:
  1. a **shared strain library** across dispensaries (generate a strain's public facts once; per-dispensary data — links,
     sizes, lab data — stays separate; must respect the DB-per-owner isolation plan, so only public strain facts are shared);
  2. **Anthropic batch pricing** (50% off for non-urgent jobs) fed by the same free research bundles;
  3. generate the **first big batch inside Claude Code** (subscription instead of API cost; our code prepares each strain's
     research free, results are written to the DB) — suits the first dispensary, not self-service SaaS;
  4. a **cheaper model** for easy strains (Sonnet was thinner in an early test; re-test on the 17 research cases first).
  Current cost per strain 5-24c (a 150-product menu ~= $8-35). Keep testing a few brands with the saved test cases.
- ~~Sync from Live redesign / Flagged Queue~~ — **done 2026-09-25: both removed.** Refresh live menu + click-to-link replace them (see below).

## Other open items
- Cautions phrase list needs legal/compliance review (still a draft by Claude).
- `dev` / `main` branch split (everything is on `main` today); production checklist: prod `.env`, session secret, decide the
  Basic Auth wall, migrations on the prod DB, remove test data/demo login.
- Edibles product type (Grasslandz etc.); Marawanna / Redwood County Weed Co / brand websites still blank.
- Customer-facing live menu tile (reads profiles through the Sweed links); Product History / Menu Editor / Education tiles.
- Sold-out products: Sweed's public storefront drops them entirely; a fuller Sweed API (back-office key) would show them as
  inactive -- ask the Sweed rep (see incident notes).
- Deep-search button with spend tiers (idea from earlier; the Misc search covers the main need).

## Done 2026-09-25
- **Edit screen** on every strain card (text fields, comma lists, Cautions as the fixed approved checkboxes); each changed field is marked
  "edited" and a re-run of the AI (`redo`) leaves it alone (`strains.edited_fields`; verified with the AI stubbed).
- **Refresh live menu** (`POST /api/sync/refresh`, ~20-30s): fetches the menu once; updates each linked product's on/off-menu flag, sizes and
  prices; strain = Active if any linked product is on the menu else Not active; lists menu products with no profile (43 on the menu, 38 with no
  profile at test time). Never creates strains or flags, never overwrites text. An empty pull changes nothing (protects against a bad fetch).
- **Flagged Queue removed entirely** (page, route, `/api/flags`, `sync_flags` table + 3 enums, `dispensaries.sync_review_mode` + enum) and the **old
  Sync from Live** (`run_sync.py`, `POST /api/sync/`, its button, its nginx location) -- they only existed to feed the flag queue. Migration
  `b8d3f0a6c512` is reversible (round-trip tested); the table was empty. The Product History tile is still a placeholder reading `sync_log`
  (nothing writes to it now; Refresh could log snapshots when that tile is built).
- Link to live menu is the replacement for the old sync's match suggestions: the app finds candidate products (name, type, brand, sizes) and one
  click connects them (never silent auto-link, per the project rule).
