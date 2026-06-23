# Cron Jobs and journalctl

## What it is

**cron** is Linux's built-in task scheduler. It runs commands automatically at a time you specify — no code running continuously, no external service needed. You define jobs in a file called the **crontab** (cron table), one line per job.

**journalctl** is the tool for reading the system journal — the central log where cron output, systemd services, and anything that writes via `logger` ends up.

---

## How cron timing works

Each cron line has five time fields followed by the command:

```
minute  hour  day  month  weekday  command
  0      15    *     *       *     /usr/local/bin/legit-buddy-scrape
```

- `*` means "every" — so the above runs at minute 0, hour 15, every day
- **Times are in UTC.** The server clock is UTC. CST is UTC−6, CDT is UTC−5.
  - 9:00 AM CST = 15:00 UTC → `0 15 * * *`
  - 10:00 PM CST = 04:00 UTC (next day) → `0 4 * * *`

Edit with: `crontab -e` (root's crontab). View with: `crontab -l`.

---

## How it's set up on this server

Root's crontab runs four scrapes per day for legit-buddy:

```
0 15 * * *  /usr/local/bin/legit-buddy-scrape   # 9:00 AM CST
0 19 * * *  /usr/local/bin/legit-buddy-scrape   # 1:00 PM CST
30 22 * * * /usr/local/bin/legit-buddy-scrape   # 4:30 PM CST
0 4 * * *   /usr/local/bin/legit-buddy-scrape   # 10:00 PM CST
```

The script at `/usr/local/bin/legit-buddy-scrape`:
1. Loads `.env` (so Python scripts get the API keys)
2. Runs `pipeline/scraper.py` — fetches products from Sweed, writes `docs/products.json`
3. Runs `pipeline/enrich.py` — calls Claude API for any new strains only

---

## Reading cron output with journalctl

Cron itself doesn't show output anywhere obvious. The script pipes output to `logger`:

```bash
"$PYTHON" pipeline/scraper.py 2>&1 | logger -t "legit-buddy-scrape"
```

`logger` writes to the system journal with the tag `legit-buddy-scrape`. Read it with:

```bash
journalctl -t legit-buddy-scrape --since today       # today's runs
journalctl -t legit-buddy-scrape -n 100              # last 100 lines
journalctl -t legit-buddy-scrape --since "2026-06-23 15:00"  # specific run
```

For systemd services (not cron), use `-u` instead of `-t`:

```bash
journalctl -u legit-buddy -f       # live logs for the legit-buddy service
journalctl -u nginx --since today  # nginx logs today
```

---

## What cron does NOT do

- **No alerts on failure** — if your script crashes, cron doesn't email you or notify anything (unless you configure `MAILTO` in the crontab, which we have not). Check the journal manually if something looks wrong.
- **No retry** — if the 9 AM run fails, it doesn't try again. The next run is 1 PM.
- **No environment** — cron starts with a minimal shell environment. Always use full paths (`/usr/local/bin/python`, not `python`) and explicitly load `.env` if your script needs env vars.
- **CORS doesn't protect server endpoints** — completely unrelated concept, but worth noting here: CORS only restricts browsers. Cron jobs and any script using curl or requests bypass it entirely.

---

## Key commands

```bash
crontab -l                                          # view root's crontab
crontab -e                                          # edit root's crontab
journalctl -t legit-buddy-scrape --since today      # check scrape logs
journalctl -u legit-buddy -f                        # live app logs
journalctl --since "1 hour ago"                     # everything recent
```
