# Legit Buddy: Sweed API Scraper + Cron Setup

**Date:** 2026-06-23

## What happened

Migrated legit-buddy's daily menu scraper from GitHub Actions to a server-side cron job. Discovered and fixed the correct way to call the Sweed POS API — after significant debugging.

---

## The Sweed API — what it actually requires

The Sweed POS API endpoint is `POST /_api/Products/GetProductList`. It has unusual requirements:

| Requirement | Where | Value |
|---|---|---|
| `storeid` | HTTP **header** (not body) | `434` (South Metro store ID) |
| `ssr` | HTTP header | `"false"` |
| `last_store` | Cookie | URL-encoded JSON: `{"id":434,"url":"https://...","routeName":"/south-metro"}` |
| `swa_Common/isAgeChecked` | Cookie | `"true"` |

**Why the header, not body?** Discovered by capturing browser Network tab headers. The API returns 400 "storeId required" if the header is missing — even if storeId is in the request body. Sweed's server-side validation looks in headers only.

The flower/pre-roll/edible categories each have a numeric ID (`5221`, `5222`, etc.) that goes in the POST body under `filters.category`.

### THC value fix

The API returns THC as an array: `[0.028, 16.8]`. These are delta-9 THC (raw) vs total THC (THCA converted, what customers actually see). Fixed scraper to use `max()` instead of `[0]` so the menu shows total THC (16.8% not 0.028%).

### Flower SSR issue

The flower category page renders server-side — no API call fires in the browser when loading. Pre-roll and edibles fire the API client-side. Scraper handles this transparently because it calls the API directly (bypasses browser rendering entirely). Per-category DOM fallback exists but is not needed for the happy path.

---

## Cron setup

### Script: `/usr/local/bin/legit-buddy-scrape`

```bash
#!/bin/bash
set -e
APP_DIR="/opt/legit-buddy-private"
PYTHON="$APP_DIR/venv/bin/python"
LOG_TAG="legit-buddy-scrape"
log() { logger -t "$LOG_TAG" "$1"; echo "$1"; }
log "=== Scrape started ==="
cd "$APP_DIR"
set -a; source "$APP_DIR/.env"; set +a
log "Running scraper..."; "$PYTHON" pipeline/scraper.py 2>&1 | logger -t "$LOG_TAG"
log "Running enrich..."; "$PYTHON" pipeline/enrich.py 2>&1 | logger -t "$LOG_TAG"
log "=== Scrape complete ==="
```

Both Python commands pipe to `logger`, so their exit codes are masked from `set -e`. The script always completes as long as `logger` works. This is acceptable — error details still appear in the journal.

### Crontab (root)

```
0 15 * * *  /usr/local/bin/legit-buddy-scrape   # 9:00 AM CST
0 19 * * *  /usr/local/bin/legit-buddy-scrape   # 1:00 PM CST
30 22 * * * /usr/local/bin/legit-buddy-scrape   # 4:30 PM CST
0 4 * * *   /usr/local/bin/legit-buddy-scrape   # 10:00 PM CST
```

Cron times are UTC. Store is CST (UTC−6). Check logs with:
```bash
journalctl -t legit-buddy-scrape --since today
```

### Error handling behavior

| Scenario | What happens |
|---|---|
| 0 products scraped | Keeps existing `products.json` — menu stays up with previous data |
| Python exception | Logs to journal, script continues to enrich step (pipe masks exit code) |
| enrich.py fails (bad API key) | Skips unenriched strains only; existing entries never overwritten |
| Silent failure | No alert — check journal manually if menu looks stale |

---

## What was disabled

- `legit-buddy-private` GitHub Actions workflow (ID 299817787) disabled — cron replaced it
- `legit-buddy-api` (public site, GitHub Pages) left **untouched** — it has its own workflow that must stay active

---

## What I learned

- Sweed's API uses a custom `storeid` HTTP header, not the request body — this is unusual and not documented anywhere
- Cron runs UTC; always convert to local timezone when scheduling
- `logger -t TAG` sends output to system journal; retrieve with `journalctl -t TAG`
- Piping to `logger` masks the original command's exit code from `set -e` — the pipe's exit status is logger's
- Server-side rendered (SSR) pages don't fire API calls in the browser; scraping the API directly avoids this entirely

---

## Follow-up

- Add real `ANTHROPIC_API_KEY` to `/opt/legit-buddy-private/.env` — enrich.py currently skips new strains without it
- Edibles THC values showing 101%/91.6% — likely mg/unit reported as %, worth investigating once API key is in place
- Next: Phase 3 frontend decoupling (Claude Code Web)
