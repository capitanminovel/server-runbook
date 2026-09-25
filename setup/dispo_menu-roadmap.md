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
- **Sync from Live redesign** (greyed out "In Development"): decide what it should do once Refresh live menu exists.
- **Flagged Queue**: recommendation = grey out like Sync until the sync is redesigned (most of it overlaps with click-to-link
  and the "menu products without a profile" list). Not decided.

## Other open items
- Cautions phrase list needs legal/compliance review (still a draft by Claude).
- `dev` / `main` branch split (everything is on `main` today); production checklist: prod `.env`, session secret, decide the
  Basic Auth wall, migrations on the prod DB, remove test data/demo login.
- Edibles product type (Grasslandz etc.); Marawanna / Redwood County Weed Co / brand websites still blank.
- Customer-facing live menu tile (reads profiles through the Sweed links); Product History / Menu Editor / Education tiles.
- Sold-out products: Sweed's public storefront drops them entirely; a fuller Sweed API (back-office key) would show them as
  inactive -- ask the Sweed rep (see incident notes).
- Deep-search button with spend tiers (idea from earlier; the Misc search covers the main need).
