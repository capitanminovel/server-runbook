# legit-buddy removed (2026-09-27)

No longer used (user decision). Removed to free RAM for the dispo_menu demo site.

- **Removed:** `legit-buddy.service` (FastAPI on port 8002), its nginx site (it answered requests to the bare IP),
  the 4x-daily scrape cron lines and `/usr/local/bin/legit-buddy-scrape`, and `/opt/legit-buddy-private` (253 MB).
- **Backup:** `/root/backups/legit-buddy-private-2026-09-27.tar.gz` (root-only; code without venv, `SESSION_NOTES.md`,
  its `.env`, the unit and scrape script) plus `root-crontab-before-legit-buddy-removal.txt`. Code is also on GitHub
  (capitanminovel/legit-buddy-private).
- **Its Anthropic key is the same key dispo_menu uses** -- still valid, nothing to rotate for dispo.
- **Open (needs your OK):** with legit-buddy's site gone, the bare IP falls through to nginx's `default` site, which
  serves `/var/www/html` -- including an old Pi-hole web-interface checkout in `/var/www/html/admin` (public source
  code, no secrets). Proposed fix: in `sites-available/default`, `location / { return 444; }` (close the connection for
  unknown host names). Blocked by the permission system when I tried; left for you.
