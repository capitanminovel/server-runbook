# dispo_menu demo site: legitdemo.withcapitan.com (set up 2026-09-27)

For Legit's admin team to use now. It's a **separate copy** of dispo_menu: same code as dev, but its own database,
service user, files and settings, so dev work and tests can't break what the team is using.

## What's different from dev
| | Dev | Demo |
|---|---|---|
| Address | dispo-admin / dispo-api / dispo-menu `.dev.withcapitan.com`, basic auth | `legitdemo.withcapitan.com`: admin at `/`, API at `/api` (same address, no CORS); app login only |
| Database | `dispo_menu_dev` (login `dispo_menu`) | `dispo_menu_demo` (login `dispo_demo`, can't connect to dev's DB) |
| Service user | `dispo` | `dispo-demo` |
| Code / venv | `/opt/dispo-menu` | `/opt/dispo-menu-demo` |
| Data / uploads / backups | `/var/lib/dispo-menu` | `/var/lib/dispo-menu-demo` |
| API | `dispo-menu-api`, port 8003 | `dispo-menu-demo-api`, port 8004 |
| Jobs | `dispo-menu-{refresh,health,backup}` | `dispo-menu-demo-{refresh,health,backup}` (generated from dev's units at deploy) |
| COA lab reports | on | **off** (`FEATURE_COA_FILES=false`) |
| AI generations | no cap | **10** (`AI_GENERATION_LIMIT=10`; new profiles + Redos; refused before any paid call) |
| Customer menu / Kiosk | yes | **hidden**; nginx returns 404 for `/api/public/` until the legal review |

## How it was set up
- Data: `pg_dump` of dev (38 strains, brands, trainings, schedule, research sites) restored as `dispo_demo`; dev's logins
  removed; one admin **admin@legitdemo.com** created with a random password (initial password:
  `/root/backups/.demo-admin-initial-pw`, root-only; change it after first login). AI-generation count reset to 0.
  Dispensary renamed "Legit Cannabis (demo)". Uploads (13 MB) copied.
- `.env`: `/opt/dispo-menu-demo/api/.env` (root:dispo-demo 640, not in git): own `SESSION_SECRET` (dev cookies don't
  work here), own DB login, same Anthropic key and Sweed settings as dev.
- nginx: `/etc/nginx/sites-available/legitdemo.withcapitan.com` + `snippets/dispo-demo-proxy.conf`; same rate-limit
  zones as dev; `X-Content-Type-Options`, `X-Frame-Options DENY`, `Referrer-Policy`. TLS by certbot (auto-renews).
- Postgres hardening done at the same time: `REVOKE CONNECT ON DATABASE dispo_menu_dev FROM PUBLIC` (by default any
  login could connect to any database).

## Deploying
`./deploy.sh demo` from `/root/Dispo_menu`. It is **never** part of `./deploy.sh` (all), so dev work reaches the team
only when you choose. It syncs the code, installs requirements if they changed, runs migrations as `dispo-demo`,
installs/refreshes units, restarts, and builds the admin with an empty API base URL and no menu URL.

## Things to know
- **RAM:** the droplet has 1 GB. The second API process fits but leans on swap (about 700 MB of swap in use after setup).
  Moving to the 2 GB droplet would make both copies comfortable.
- The Anthropic key is shared with dev: the demo's 10-generation cap is the spend guard. Deleting a strain doesn't give
  a generation back.
- The demo's 5-minute refresh also calls Sweed, so the store's menu is now read by both copies.
- No headless-browser fallback on the demo (plain HTTPS Sweed reads only).

## Key commands
```bash
systemctl status dispo-menu-demo-api
journalctl -u dispo-menu-demo-api -f
systemctl list-timers | grep demo
sudo -u postgres psql -d dispo_menu_demo -c "select count(*) from ai_generations"   # generations used
```

## Update 2026-09-28
- Deployed the "hard-to-find strains" work to the demo: Also known as, Pages to read, lineage check, automatic
  other-name search on thin research, Thin research box, Redo profile, "What's new" tip. `FEATURE_NAME_SEARCH=true`,
  `NAME_SEARCH_LIMIT=20` (manual button; the automatic search only runs inside a Generate/Redo, so the 10-generation cap
  covers it). nginx: `find-other-names` rate-limited like generating. **No `SHOW_COSTS`** -> no prices anywhere
  (checked in a browser, including confirm pop-ups).
- AI-generation count reset to 0 of 10 (user request).
- Thin demo strains re-researched **in a Claude Code session** (free, `scripts/session_generate.py reresearch/update`,
  same Generator research code): Double Sour Grape (aka Double Grape), Sunset Tea (aka Sun Tea), Burger Breath (Atlas
  Seed pages), Terp Poison (GTR Seeds pages). Updated in place on demo + dev; live-menu links kept.
- Guides updated (team site address, section 4.2 hard-to-find strains, 10-generation limit). Team email template:
  `docs/guides/email-team-update-2026-09-28.md`. Open items: `setup/dispo_menu-review-list.md`.

## Update 2026-09-29: customer menu + kiosk preview
- **legitdemo-menu.withcapitan.com** (+ `/kiosk`): nginx `sites-available/legitdemo-menu.withcapitan.com`, basic auth
  (`/etc/nginx/.htpasswd-dispo-demo-menu`, user `legit`; password given to the user, not written here),
  `X-Robots-Tag: noindex` + robots.txt, CSP `connect-src 'self'`. `/api/public/` is proxied to the demo API on this
  host only (the main demo site still returns 404), so the password covers the data too. TLS by certbot.
- `deploy.sh demo` builds the customer menu (empty API base URL, `VITE_DISPENSARY=dev-dispensary`) into
  `/var/www/dispo-demo-menu`, and the admin's menu/kiosk tiles now point there.
- Minnesota compliance on the menu: see `concepts/mn-cannabis-advertising-rules.md`.
- **Shared staff login (2026-09-29):** `team@legitdemo.com`, role **employee** (view-only; server-enforced — generate/redo
  return 403). Password given to the user, not written here. Admins keep `admin@legitdemo.com`; individual logins are
  still recommended (review list).

## Update 2026-09-30: "Dispensary Tool", demo framing, what's still missing
- User-facing name is now **Dispensary Tool** (was "Dispo Tool"): app, browser tab, home-screen name, guides
  (`Dispensary-Tool-*.docx`), emails. Internal names (dispo_menu, dispo-menu-* services, hostnames) unchanged on purpose.
- The demo is framed as **a demo of the idea for the team, to find gaps**: `SITE_NOTICE` in the demo `.env` shows a
  banner on the dashboard; guides have "About this demo"; the "10-minute demo" section is now "A quick tour" (a live
  walkthrough comes later); the manager/admin email (`docs/guides/email-admin-launch.md`) asks for feedback.
- **Refresh live menu** now shows coverage ("39 of 44 menu products have a profile") and a **No profile yet** checklist
  grouped by brand, each with **Generate** that pre-fills the Generator (name without PR/pack size, brand, type).
- Removed the duplicate demo profile #187 "Grape Zkittlez" (the team re-made it as #188 Grape Canyon Zkittles).
