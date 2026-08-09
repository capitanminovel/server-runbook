# dispo_menu — Real Admin Login + Dashboard

**Date:** 2026-08-09

## What this is

Follow-up to `dispo_menu-foundation-2026-08-02.md`. Built the actual login → dashboard → tile
flow in the admin app, which required resolving a real architecture question the original docs
didn't answer: how does the admin app know which dispensary (tenant) a logged-in user belongs to?

## Decision: admin tenant resolution comes from the user's account, not the subdomain

The docs describe subdomain-based tenant resolution (`dispensary-name.dispomenu.com` → look up
`dispensary_id`) but that's specified for the **customer-facing menu**, which has no login step —
the URL is the only signal for who the customer is looking at. The **admin app** is different:
there's already a login step, so the natural place to get `dispensary_id` from is the matched
`staff_users` row after authenticating — no subdomain parsing needed. One shared admin login URL
works for every tenant; `dispensary_id` comes off whichever `staff_users` row the email matched.

## Bigger question surfaced along the way: tenant isolation model

While working through this, re-examined a bigger assumption: is one shared Postgres database
(current state, `dispensary_id` on every row) actually the right call, or should each tenant get
a separate database? At ~10 real dispensaries (not 100+), the tradeoff looks different than
generic SaaS advice would suggest — with a single shared DB, tenant isolation depends entirely on
every query correctly filtering by `dispensary_id`; a missed filter is a real cross-tenant data
leak, not just a bug, which matters more than usual for a compliance-sensitive industry.

Landed on a **three-tier plan**, not yet built beyond the first tier:

1. **Now:** single shared database (current state) — cheapest to keep building the foundation on.
2. **Target:** **database-per-owner**, not per-dispensary. An "owner" (e.g. the same person
   running both `dinky-dope-menu` and `legit-cannabis-south-metro-menu` today) gets one database;
   their individual stores are rows in that database's `dispensaries` table, still scoped by
   `dispensary_id` exactly as already built. Different *owners* get structural isolation (separate
   databases, a query bug literally cannot cross that boundary); stores under the *same* owner
   share a database because that owner legitimately wants shared staff/suppliers/reporting across
   their own locations.
3. **Not built yet, deliberately:** the control-plane piece (a small shared lookup table mapping
   subdomain → which owner's database to connect to, plus dynamic per-request DB connection
   routing in `app/db.py`). Deferred until a second owner actually exists — building and testing
   multi-owner routing against only one real owner would be guesswork. Today's single database
   *is* that one owner's database, informally, with no code change needed to represent that.

**Follow-up:** build the control-plane + per-owner connection routing when dispensary #2's owner
(a different person/company than the current one) actually signs up — not before.

## Schema fix: sync_review_mode was implicitly global

The original docs described the manual/auto sync review toggle as "site-wide," which read as one
global setting for the whole app — wrong for multi-tenant, since two different dispensaries might
legitimately want different behavior. Added `dispensaries.sync_review_mode` (enum `manual`/`auto`,
default `manual`) via migration `300c2c0c8b1b`. Gotcha hit along the way: Alembic's
`--autogenerate` doesn't create a new Postgres enum type for an `ADD COLUMN` on an existing table
(only for a fresh `CREATE TABLE`) — had to add `review_mode_enum.create(op.get_bind())` explicitly
in the migration's `upgrade()`, and `.drop()` in `downgrade()`.

## What was built

**Backend (`apps/api`):**
- `app/auth/passwords.py` — bcrypt hash/verify helpers
- `app/auth/session.py` — `get_current_staff_user` now does a real session → DB lookup instead of
  raising `NotImplementedError`
- `app/routes/auth.py` — real `/api/auth/login` (verifies password, sets session), `/me`, `/logout`
- CORS added (`app/main.py`, `app/config.py`) — admin/customer-menu frontends are on different
  subdomains than the API, so credentialed cross-origin requests need an explicit origin allowlist
  (wildcard `*` doesn't work once `allow_credentials=True`)
- `scripts/seed_dev.py` — one-off script creating a dev dispensary + staff user, since there's no
  real onboarding/registration flow yet. Ran once: created "Dev Dispensary" (id=1) and
  `admin@dispo-menu.dev`. Credentials given to the user directly in chat, not committed anywhere.

**Frontend (`apps/admin`):**
- `react-router-dom` added; routes for `/`, `/dashboard`, `/strains`, `/history`, `/menu-editor`,
  `/education`
- `AuthContext` (checks `/api/auth/me` on load) + `ProtectedRoute` (redirects to `/` if not
  logged in)
- Login page wired to the real endpoints; dashboard shows a 5-tile grid per the site map —
  Strain List gets a real page shell (matches the UI section of `strain-list-tile-spec.md`, no
  live data yet since the backend endpoints behind it are still stubs), the other 3 internal
  tiles are "coming soon" placeholders, Customer Facing Page links out to the customer-menu app

Verified end-to-end against the live preview deployment (not just locally): CORS preflight,
login setting a real session cookie, `/me` authenticating via that cookie across the
`dispo-admin` → `dispo-api` subdomain boundary, and the rebuilt static bundle serving correctly
behind Basic Auth.

## Follow-up

- [ ] Build the control-plane (subdomain/owner lookup + per-owner DB connection routing) once a
      second owner actually exists — see "tenant isolation model" above
- [ ] Decide if Misc field is AI-fillable or admin-only (still open from the original docs)
- [ ] Decide if COA-tested status is customer-visible or admin-only (still open from the original docs)
- [ ] Revisit `staff_users` field list once real auth is more fleshed out (e.g. password reset,
      invite flow — none of that exists yet, just login)
- [ ] Wire the Strain List page to real data once the backend's strains/sync endpoints are built
      (next session, per the original plan)
