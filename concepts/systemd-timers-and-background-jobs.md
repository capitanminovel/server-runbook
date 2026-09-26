# systemd timers and background jobs

## What it is
A **systemd timer** is systemd's version of a cron job: a small `.timer` file that says *when*, paired with a `.service`
file that says *what to run*. When the timer fires, systemd starts the service; the service runs once
(`Type=oneshot`) and exits.

For dispo_menu we use three:

| Timer | Runs | What it does |
|---|---|---|
| `dispo-menu-refresh.timer` | every 5 min (`*:0/5`) | auto "Refresh live menu" — only inside store hours |
| `dispo-menu-health.timer` | every 5 min (`*:2/5`, offset so they don't start together) | is everything up? opens/clears developer alerts |
| `dispo-menu-backup.timer` | 3:30 AM Chicago time | database dump + copy of uploaded files |

## Why it matters / how it works
**Why not cron?** Both work (the legit-buddy scraper uses cron — see `cron-and-journalctl.md`). Timers were a better fit here because:
- **Same sandbox as the API for free.** The `.service` file carries the same hardening (`User=dispo`, `ProtectSystem=strict`, only `/var/lib/dispo-menu` writable…). A cron job would run as whoever owns the crontab, usually with far more access.
- **No overlap.** If a run is still going when the timer fires again, systemd doesn't start a second copy. With cron, a slow run plus the next one can pile up.
- **Logs in one place.** `journalctl -u dispo-menu-refresh` shows every run, with timestamps, like the API's logs.
- **Time zones.** `OnCalendar=*-*-* 03:30:00 America/Chicago` means 3:30 AM *store* time, even though the server clock is UTC and even across daylight-saving changes.
- **Missed runs.** `Persistent=true` on the backup: if the server was off at 3:30, it runs as soon as it's back.
- **Timeouts.** `TimeoutStartSec=` kills a job that hangs instead of letting it sit forever.

**Why separate processes from the website?** Each job is its own short-lived program (`python -m app.jobs.<name>`). If the
menu refresh hangs on a slow Sweed response or crashes, the website (the `dispo-menu-api` service) doesn't notice — it's a
different process. They share the code and the database, not the running program. `Nice=10` also tells Linux to let the
website go first when both want the CPU.

## How it's set up on this server
- Unit files live **in the repo**: `Dispo_menu/ops/systemd/*.service|*.timer`. `./deploy.sh api` copies any changed ones to
  `/etc/systemd/system/`, runs `systemctl daemon-reload`, and makes sure each timer is enabled.
- Code: `apps/api/app/jobs/` (one file per job, shared helpers in `_common.py`).
- Settings come from the same `/opt/dispo-menu/api/.env` as the API (`MENU_REFRESH_HOURS`, `STORE_TIMEZONE`, `BACKUP_*`, `SMTP_*`…).
- Each job records its last run / last success / failures-in-a-row in the `job_status` table.

## What it does NOT do
- A timer doesn't retry a failed run by itself — the next scheduled run is the retry. (The menu refresh adds its own
  back-off after 3 failures: 10, 20, 40, then every 60 min.)
- It can't tell you the whole server is down — nothing on the server can. That needs an outside monitor (e.g. UptimeRobot hitting `https://dispo-api.dev.withcapitan.com/health`).
- Editing a unit file in `/etc/systemd/system` by hand gets overwritten by the next deploy — edit `ops/systemd/` in the repo.

## Key commands
```bash
systemctl list-timers 'dispo-menu-*'            # when each one runs next / last ran
journalctl -u dispo-menu-refresh -n 30          # last runs of one job (also: -health, -backup)
journalctl -u 'dispo-menu-*' --since today      # everything dispo_menu logged today
systemctl start dispo-menu-backup.service       # run a job right now, by hand
systemctl status dispo-menu-health.service      # did the last run succeed?
systemctl disable --now dispo-menu-refresh.timer   # pause a job (deploy.sh turns it back on)
systemd-analyze calendar '*-*-* 03:30:00 America/Chicago'   # check what an OnCalendar line means
cd /opt/dispo-menu/api && sudo -u dispo ../venv/bin/python -m app.jobs.show_alerts   # open alerts + job status
```
