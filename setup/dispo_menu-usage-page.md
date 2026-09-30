# dispo_menu developer usage page (set up 2026-09-30)

**https://dispo-admin.dev.withcapitan.com/usage/** — behind the dev site password (the team doesn't have it).
Updates every 10 minutes; the page reloads itself every 5.

## What it shows (Demo first, then Dev)
- **Usage:** AI generations used vs the limit, other-name searches, AI cost (24 h / 7 days / all time, estimates from
  token counts), every AI run (when, generate/redo/name search, strain, cost).
- **Who's using it:** each login's last sign-in, sign-ins in 7 days / all time, failed attempts. Plus failed attempts
  with unknown emails, and a red card if there are 10+ failed logins in 24 h (password guessing).
- **Team updates (7 days):** profiles created / edited (which fields) / linked to the live menu, trainings, schedules,
  new logins.
- **Health:** API up right now, background jobs (refresh, health, backup), open alerts, which commit is deployed.

## How it works (and why it's built this way)
- The app records two new things: `login_events` (success/failed, the account if it's a real one — **no IP addresses,
  no typed emails**) and `ai_generations.cost_usd`.
- `ops/monitor/usage_report.py` runs as its own system user **`dispo-monitor`** from its own venv
  (`/opt/dispo-monitor`), sandboxed like the app units, and can only write `/var/lib/dispo-monitor/usage/`.
- It reads both databases with its own Postgres login **`dispo_monitor`**, which has **column-level SELECT only**
  (e.g. `staff_users` email/role but never `password_hash`; strain names/status but not the content). It can't change
  anything. Credentials: `/etc/dispo-monitor/env` (root:dispo-monitor 640).
- It writes one static HTML page (swapped in atomically). A problem with the monitor can't affect the app.
- `deploy.sh` records what's deployed where in `/var/lib/dispo-monitor/versions.json` (commit, time, the demo's limits).
- nginx: `location /usage/` in the dev admin site (alias, `no-store`, `noindex`, strict CSP). The team site's
  `/usage/` is just its own app page, not this.

## What it does NOT do
- It doesn't send alerts (email alerts still aren't configured). Check the page, or ask Claude to read it.
- Costs are estimates; the Anthropic console is the source of truth. Runs from before 2026-09-30 have no cost.
- Logins from before 2026-09-30 weren't recorded.

## Key commands
```bash
systemctl start dispo-monitor.service        # regenerate now
journalctl -u dispo-monitor -n 20            # if the page looks stale
./deploy.sh monitor                          # install/refresh (from /root/Dispo_menu)
# new table the page should read? grant it to the read-only login, per database:
sudo -u postgres psql -d dispo_menu_demo -c "GRANT SELECT (col1, col2) ON some_table TO dispo_monitor"
```

## History
- 2026-09-30: demo AI usage reset to 0/10 (and 0/20 name searches) at the user's request; the 7 earlier rows are in
  `/root/backups/demo-ai-generations-before-reset-20260930.jsonl`.
