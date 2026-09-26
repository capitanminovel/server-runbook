# dispo_menu — auto menu refresh, health checks, backups, developer alerts (2026-09-26)

How the background jobs work: `concepts/systemd-timers-and-background-jobs.md`.

## Auto "Refresh live menu" — every 5 minutes, store hours
- Timer `dispo-menu-refresh.timer`, code `app/jobs/menu_refresh.py`. Store hours: `.env` `MENU_REFRESH_HOURS=08:00-22:00`,
  `STORE_TIMEZONE=America/Chicago`. **Confirm the real hours with Legit** and change `.env` (no code change; next run picks it up).
- Does what the button does: on/off menu, sizes/prices, Active / Not active, brand snapshot. Never links anything.
- **Never starts the headless browser.** If Sweed's direct call fails, the run fails. After 3 failures in a row (15 min),
  it opens an alert and backs off (10 → 20 → 40 → 60 min between tries); the first success clears it.
- Cost: ~1 s and ~0.1 s CPU per run; ~170 runs/day ≈ 6 small requests each to Sweed's storefront.

## Health check — every 5 minutes
`app/jobs/health_check.py`. Each check opens an alert when wrong and clears it when fine:

| Alert key | Means | First thing to run |
|---|---|---|
| `api_down` | API not answering on 127.0.0.1:8003 | `journalctl -u dispo-menu-api -n 50` |
| `api_public_down` | `https://dispo-api…/health` fails (nginx / TLS / DNS) | `systemctl status nginx`, `nginx -t` |
| `admin_site_down` | admin site not 200/401 | same |
| `tls_cert:<host>` | HTTPS cert expires in < 14 days | `systemctl list-timers certbot`, `certbot renew --dry-run` |
| `disk_space` | disk ≥ 85 % | `du -sh /var/lib/dispo-menu/*`, `df -h` |
| `low_memory` | < 60 MB available | `ps aux --sort=-rss \| head` |
| `backup_stale` | newest DB backup > 26 h old | `journalctl -u dispo-menu-backup` |
| `menu_refresh_stale:<id>` | no successful refresh in 20 min during store hours (timer stopped?) | `systemctl list-timers 'dispo-menu-*'` |
| `menu_refresh:<id>` | Sweed refresh failed 3× in a row | `journalctl -u dispo-menu-refresh -n 30` |
| `backup` | the backup job itself failed | `journalctl -u dispo-menu-backup` |

**Gap:** if the whole droplet is down, nothing on it can alert you. Add a free outside monitor (UptimeRobot etc.) on
`https://dispo-api.dev.withcapitan.com/health` (returns `{"status":"ok"}`).

## Developer alerts — email is OFF until set up
`app/alerts.py`. One email when a problem starts, a reminder every `ALERT_REMINDER_HOURS` (6) while it lasts, one
"RESOLVED" when it clears — never one per 5-minute run. Until email is configured, alerts go to the journal
(`journalctl -u 'dispo-menu-*' | grep ALERT`) and the `dev_alerts` table:
```bash
cd /opt/dispo-menu/api && sudo -u dispo ../venv/bin/python -m app.jobs.show_alerts
```

**To turn email on (Gmail example)** — add to `/opt/dispo-menu/api/.env` (NOT the repo):
```
ALERT_EMAIL_TO=capitanminovel@gmail.com
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=<sending gmail address>
SMTP_PASSWORD=<Google "app password" — Google Account → Security → 2-Step Verification → App passwords>
SMTP_FROM=<sending gmail address>
```
Never the real Gmail password: an app password only sends mail and can be revoked on its own. Mail goes over STARTTLS
(encrypted). Nothing to restart — each job reads `.env` when it runs. Test: stop the API for 6 minutes, or run the
health check with a bad URL. **Why a separate sending account is nicer:** if its app password ever leaks, your main inbox isn't exposed.

## Backups — nightly 3:30 AM store time
`app/jobs/backup.py` → `/var/lib/dispo-menu/backups/` (owned by `dispo`, mode 700):
- `db-YYYYMMDD-HHMM.dump` — `pg_dump` custom format, verified with `pg_restore --list`, kept 14 days (`BACKUP_KEEP_DAYS`).
  The DB password is passed via `PGPASSWORD`, not the command line (command lines are visible to everyone in `ps`).
- `uploads/` — a mirror of uploaded files. Uploads never change once saved, so only new files are copied; a file deleted
  in the app moves to `uploads/.deleted/` and is purged after 14 days (an accidental delete can be undone for 2 weeks).
- **Limit:** same disk as the live data → protects against mistakes (bad migration, deleted training), not against
  losing the droplet. **Turn on DigitalOcean droplet backups** (Droplet → Backups; ~20 % of the droplet price) for off-server copies.

### Restore (tested 2026-09-26 into a scratch database)
```bash
ls -lt /var/lib/dispo-menu/backups/db-*.dump | head -3          # pick one
systemctl stop dispo-menu-api dispo-menu-refresh.timer dispo-menu-health.timer
# the database is dispo_menu_dev (see DATABASE_URL in /opt/dispo-menu/api/.env). The backup folder is readable only by
# 'dispo', so root feeds the file in with '<' rather than giving postgres the path:
sudo -u postgres pg_restore --clean --if-exists --no-owner --role=dispo_menu -d dispo_menu_dev < /var/lib/dispo-menu/backups/db-XXXX.dump
systemctl start dispo-menu-api dispo-menu-refresh.timer dispo-menu-health.timer
# a deleted upload: find it under backups/uploads/.deleted/<dispensary_id>/ and copy it back to /var/lib/dispo-menu/uploads/<dispensary_id>/
```
Tested: `sudo -u postgres createdb restore_test && sudo -u postgres pg_restore --no-owner -d restore_test < <dump>` → strains 6/6, trainings 1/1 matched live; then `sudo -u postgres dropdb restore_test`. Practise this now and then — a backup you've never restored is a hope, not a backup.

## Other changes this day
- **API docs pages off:** `/docs`, `/redoc`, `/openapi.json` now 404 on the server (they listed every endpoint — a free map
  for attackers). Local only: `API_DOCS=true`. `/health` no longer says which environment it is.
- **Schedule tile:** one PDF/picture per post with a date range; opens on the current one. Shares upload safety with
  Education (`app/uploads.py`), same quota. nginx upload location now `^/api/(education/trainings/[0-9]+/files|schedule/)$`.
- **nginx upload limit made POST-only** (new zone `dispo_upload_post`, keyed by the same POST-only map as `dispo_generate`).
  Found by the schedule test: `GET /api/schedule/` (the list) shares the upload URL, so page loads used up the upload
  allowance and uploads started failing ("Failed to fetch" — a 503 from nginx has no CORS headers, so the browser hides it).
  Same class of bug as the 2026-09-25 strain-list fix; new zone name because nginx won't reload a zone whose key changes.
- **Python packages updated** (0 known vulnerabilities before and after): uvicorn 0.32→0.54, alembic 1.14→1.20,
  psycopg 3.2→3.3.6, pydantic-settings 2.6→2.15, anthropic 1.0→1.8, SQLAlchemy 2.0.36→2.0.54, bcrypt 4.2→4.3.
  Held back on purpose: SQLAlchemy 2.1 (removes deprecated APIs), bcrypt 5 (errors on >72-byte passwords), Playwright (must match installed Chromium).
- Tests added: `scripts/jobs_check.py` (13 checks), `scripts/schedule_check.py` (17 checks).
