# dispo_menu — Foundation Scaffold

**Date:** 2026-08-02

## What this is

`dispo_menu` is a new multi-tenant SaaS for cannabis dispensaries — AI-assisted strain profile
generation, synced against a live Sweed POS, powering a customer-facing menu and admin tools.
This was the first scaffolding session: project structure, DB schema, API route stubs, auth
shape, and frontend skeletons — no business logic yet.

**Repo:** https://github.com/capitanminovel/dispo_menu
**Path on server (dev):** `/root/Dispo_menu/`
**Docs:** `docs/dispo_menu-project-overview.md` (architecture) and `docs/strain-list-tile-spec.md`
(first tile spec) — moved from repo root into `docs/` this session to match what `CLAUDE.md`
already referenced.

## Stack decisions made this session (not fully specified in the docs, confirmed with the user)

- **Backend: Python + FastAPI**, not Node/Express as the docs' kickoff prompt originally
  suggested. Reasoning: `money_buddy` and `capitans_terps` are already FastAPI on this droplet —
  same deploy pattern (systemd + nginx), same debugging muscle memory, one less runtime to
  maintain. Pydantic also fits the Strain Generator's structured-AI-call requirement well.
- **DB access: SQLAlchemy Core (not the ORM) + Alembic.** Core query builder keeps queries
  SQL-shaped and easy to eyeball for `dispensary_id` scoping — important for a multi-tenant app
  where a missing tenant filter is a data leak, not just a bug.
- **Frontends: React + Vite + TypeScript**, both admin and customer-menu. Plain SPA, no SSR —
  neither app needs it (admin is internal login-gated tooling, customer-menu is a simple
  read-only site), and Next.js's API routes would be redundant since one FastAPI service already
  owns all API logic.
- **Auth shape: session cookies** via Starlette's built-in `SessionMiddleware` (the FastAPI
  equivalent of `express-session`) rather than JWT. Simpler for a single internal admin app on
  one domain family (`*.dispomenu.com`) — no refresh-token complexity needed yet.

## Database

Installed Postgres 16 via `apt` (was not previously installed on this droplet). Default
`listen_addresses = 'localhost'` — matches the security baseline already documented for this
project. Created a local dev role + database:

```
Role: dispo_menu
DB:   dispo_menu_dev
```

Password lives only in `apps/api/.env` (gitignored), generated via `openssl rand`.

### Schema

Six tables, defined once as SQLAlchemy Core `Table` objects in `apps/api/app/models/schema.py`
(single source of truth — also used as `target_metadata` for Alembic autogenerate):

- `dispensaries` — tenant record (subdomain, sweed slug, branding). The one table *without* a
  `dispensary_id` FK.
- `suppliers`, `strains`, `sync_log`, `sync_flags` — field-for-field from
  `docs/strain-list-tile-spec.md`, each with `dispensary_id` FK.
- `staff_users` — **not in the docs' spec** (auth fields weren't detailed yet). Added the
  minimal shape needed for the login scaffold: `id`, `dispensary_id`, `email`, `password_hash`,
  `role`, timestamps. Revisit field list once real auth is built.

Initial migration generated via `alembic revision --autogenerate` and applied
(`alembic upgrade head`) — verified all 6 tables and every FK landed correctly via `psql`.

## Project structure

```
apps/
  api/             FastAPI backend
    app/
      main.py       app entrypoint, registers routers + SessionMiddleware
      config.py     pydantic-settings, reads .env
      db.py         SQLAlchemy engine/connection dependency
      models/schema.py   Core Table definitions (see above)
      routes/       one file per tile: strains, sync, flags, suppliers, history, auth
      auth/session.py    get_current_staff_user dependency — stubbed, raises NotImplementedError
    alembic/        migrations, env.py wired to app.config + app.models.schema
    requirements.txt
    venv/           (gitignored)
    .env            (gitignored, local dev secrets)
    .env.example
  admin/            React + Vite + TS, login-gated admin app — placeholder page only
  customer-menu/    React + Vite + TS, read-only customer menu — placeholder page only
docs/
  dispo_menu-project-overview.md
  strain-list-tile-spec.md
CLAUDE.md
.gitignore          .env, venv/, node_modules/, dist/
```

All 5 route files (`/api/strains`, `/api/sync`, `/api/flags`, `/api/suppliers`, `/api/history`)
plus `/api/auth` are wired into `main.py` and confirmed reachable (`/openapi.json` shows all
paths, `/health` returns 200). Every endpoint body is a stub (`raise NotImplementedError` or
similar) — no business logic, per this session's scope.

## What I learned

- `apt install postgresql` on this droplet triggered a `whiptail` dialog about a pending kernel
  upgrade (unrelated to Postgres) that failed because apt had no interactive terminal — install
  still completed fine underneath it. Pending kernel upgrade is still outstanding
  (`/var/run/reboot-required` exists); not acted on, since rebooting is disruptive and wasn't
  asked for.
- Pointing Alembic's `target_metadata` at the same `MetaData` object the app uses for queries
  (rather than hand-writing migrations separately) means `--autogenerate` stays useful for every
  future schema change, not just this first one.

## Follow-up

- [ ] Cross-check mnlegitdev repo field choices once accessible (open item from the docs)
- [ ] Inventory actual Sweed API fields before finalizing sync mapping (open item from the docs)
- [ ] Decide Therapeutic vs. Effects distinction in the generation prompt (open item from the docs)
- [ ] Decide if Misc field is AI-fillable or admin-only (open item from the docs)
- [ ] Decide if COA-tested status is customer-visible or admin-only (open item from the docs)
- [ ] Revisit `staff_users` field list once real auth logic is built (not spec'd in docs, my addition)
- [ ] Confirm schema looks right with the user before starting the Strain Generator's AI call logic (next session)
- [ ] Set up systemd service + nginx vhost for this app once it's ready to run persistently (not done this session — this was local scaffolding only)
