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

## Dev preview deployment (added same session)

Stood up a public preview of the foundation scaffold on the existing personal domain, so it's
visible as we push — not a formal staging environment (no fake test tenant yet), just the local
dev DB and app exposed behind nginx for convenience.

**DNS:** `withcapitan.com` already has a wildcard record → `167.99.235.92`. Confirmed via
`dig` before touching anything — no new DNS records were needed, any subdomain already resolves.

**Subdomains** (the `dev.` prefix distinguishes these from the real `grow.`/`money.`/production
subdomains — signals "not a live customer-facing service"):

| Subdomain | What it serves | Port/path |
|---|---|---|
| `dispo-api.dev.withcapitan.com` | FastAPI backend (proxied) | `127.0.0.1:8003` |
| `dispo-menu.dev.withcapitan.com` | customer-menu static build | `/var/www/dispo-menu` |
| `dispo-admin.dev.withcapitan.com` | admin static build, **HTTP Basic Auth** | `/var/www/dispo-admin` |

**Why Basic Auth on admin only:** the app's own session auth (`get_current_staff_user`) is
scaffolded but not functional yet — it just raises `NotImplementedError`. Right now that's
low-risk since every endpoint behind it is a stub, but the moment real logic lands (next
session), an admin subdomain with no working auth on the public internet is a real exposure —
same pattern as the Pi-hole incident. Basic Auth at the nginx layer is a stopgap, independent of
whatever the app's real auth ends up doing. `dispo-menu` and `dispo-api` are intentionally left
open — customer-menu is meant to be public, and the API needs to be reachable by both frontends.

**Systemd service** — `/etc/systemd/system/dispo-menu-api.service`:
```ini
[Unit]
Description=dispo_menu FastAPI App
After=network.target postgresql.service

[Service]
WorkingDirectory=/root/Dispo_menu/apps/api
EnvironmentFile=/root/Dispo_menu/apps/api/.env
ExecStart=/root/Dispo_menu/apps/api/venv/bin/uvicorn app.main:app --host 127.0.0.1 --port 8003
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

**Static frontends — a permissions gotcha:** originally pointed nginx's `root` directly at
`/root/Dispo_menu/apps/*/dist`. That failed with `stat() ... Permission denied` — `/root` is
(correctly) `0700`, so `www-data` can't traverse into it at all. Rather than loosen `/root`'s
permissions (would expose everything else under `/root` to the web-facing user, not just these
two folders), copied the built `dist/` output to the conventional `/var/www/dispo-admin` and
`/var/www/dispo-menu`, owned by `www-data:www-data`, and pointed nginx there instead.

**Basic Auth credentials** — `/etc/nginx/.htpasswd-dispo-admin`, username `dispo`. Password was
generated randomly this session and given to the user directly in chat (not written to any
committed file). Rotate with `htpasswd /etc/nginx/.htpasswd-dispo-admin dispo`.

**SSL:** single `certbot --nginx -d dispo-api... -d dispo-menu... -d dispo-admin...` call — one
SAN certificate covering all three, same Certbot-managed pattern as `grow.`/`money.`. Auto-renews
via Certbot's systemd timer, same as the others.

### Manual redeploy steps (chosen over automation — foundation stage, minimize moving parts)

```bash
cd /root/Dispo_menu && git pull

# Backend — only needed if requirements.txt or app code changed
cd apps/api && source venv/bin/activate && pip install -r requirements.txt
systemctl restart dispo-menu-api

# Frontends — rebuild and re-copy to /var/www (nginx serves static files, not a live process)
cd apps/admin && npm install && npm run build && cp -r dist/. /var/www/dispo-admin/
cd apps/customer-menu && npm install && npm run build && cp -r dist/. /var/www/dispo-menu/
```

## Schema fix (added same session, before deploying the preview)

User reviewed the schema and asked me to self-critique it before we built anything on top of it.
Found and fixed three gaps via a second migration (`a4534f839b27`):

- **No indexes on any `dispensary_id` FK column.** Postgres doesn't auto-index FK columns (only
  the referenced side, via the PK) — and since every query in a multi-tenant app filters by
  `dispensary_id` first, every one of those was a full table scan. Added `index=True` on
  `dispensary_id` across `suppliers`, `strains`, `sync_log`, `sync_flags`, `staff_users`.
- **`staff_users.email` was globally unique** — wrong for multi-tenant: two different dispensary
  customers could plausibly want the same email on separate accounts (e.g. one owner running two
  locations as two tenants). Changed to a composite unique constraint on `(dispensary_id, email)`.
- **No uniqueness on `sweed_product_id` per tenant** — nothing stopped two strains in the same
  dispensary from both linking to the same Sweed product during sync. Added a partial unique
  index on `(dispensary_id, sweed_product_id) WHERE sweed_product_id IS NOT NULL`.

## Sweed + AI-generation research (mnlegitdev repo cross-check)

Read `mnlegitdev/legit-cannabis-south-metro-menu` (the live, real-data predecessor) to resolve
open items from `docs/CLAUDE.md`. Full findings in memory
(`project_dispo_menu_sweed_findings.md`); summary:

- **Sweed fields confirmed:** product `id` (stable numeric — use as `sweed_product_id`, never a
  derived hash), `name`, `brand.name`, `category.name`, `strain.prevalence.name`,
  `strain.terpenes[]`/`strain.flavors[]` (lab/COA-sourced), top-level `effects[]` (Sweed's own
  marketing tags), `variants[]` (stock/price/lab THC-CBD). Category IDs are per-store, not
  reusable across tenants. Direct API calls are WAF-blocked outside a real browser session —
  mnlegitdev's working approach uses Playwright with a `storeid` header + `last_store` cookie.
- **Therapeutic vs. Effects — resolved:** `effects` = raw Sweed tags, `therapeutic` = separate
  AI-generated field. Confirmed clean precedent to reuse.
- **Cautions — confirmed dispo_menu's stricter approach is correct:** mnlegitdev generates a
  freeform AI "negative" string; dispo_menu's fixed admin-reviewed phrase list is a deliberate
  improvement, not something to relax.
- **Bugs to design around:** product identity must never be a derived text hash (two real
  incidents there — already avoided via the real Sweed `id`); AI lineage generation hallucinates
  confidently when ungrounded — always pass known lineage as a hint; enrichment pipelines must
  never silently re-trigger on a key/schema change (a past incident nearly re-ran a paid Claude
  enrichment pass this way).

## Follow-up

- [x] Cross-check mnlegitdev repo field choices once accessible — done, see above
- [x] Inventory actual Sweed API fields before finalizing sync mapping — done, see above
- [x] Decide Therapeutic vs. Effects distinction in the generation prompt — resolved, see above
- [ ] Decide if Misc field is AI-fillable or admin-only (open item from the docs)
- [ ] Decide if COA-tested status is customer-visible or admin-only (open item from the docs)
- [ ] Revisit `staff_users` field list once real auth logic is built (not spec'd in docs, my addition)
- [x] Confirm schema looks right with the user before starting the Strain Generator's AI call logic — confirmed, plus 3 gaps found and fixed (see above)
- [x] Set up a dev preview URL — done, see "Dev preview deployment" above
