# legit-buddy removed (2026-09-27)

No longer used (user decision). Removed to free RAM for the dispo_menu demo site.

- **Removed:** `legit-buddy.service` (FastAPI on port 8002), its nginx site (it answered requests to the bare IP),
  the 4x-daily scrape cron lines and `/usr/local/bin/legit-buddy-scrape`, and `/opt/legit-buddy-private` (253 MB).
- **Backup:** `/root/backups/legit-buddy-private-2026-09-27.tar.gz` (root-only; code without venv, `SESSION_NOTES.md`,
  its `.env`, the unit and scrape script) plus `root-crontab-before-legit-buddy-removal.txt`. Code is also on GitHub
  (capitanminovel/legit-buddy-private).
- **Its Anthropic key is the same key dispo_menu uses** -- still valid, nothing to rotate for dispo.
- **Bare IP closed (approved and done the same day):** with legit-buddy's site gone, the bare IP fell through to
  nginx's `default` site, which served `/var/www/html`, including an old Pi-hole web-interface checkout in
  `/var/www/html/admin` (public source, no secrets). `sites-available/default` now has `location / { return 444; }`:
  nginx closes the connection for the bare IP or any unknown host name. Backup of the old file:
  `/root/backups/nginx-default-site.20260927`. Check: `curl -s -o /dev/null -w "%{http_code}" http://167.99.235.92/`
  prints `000`.
- **Gotcha:** right after `systemctl reload nginx`, old worker processes finish their open connections with the old
  config, so a quick re-test can still show the old answer. Test again a few seconds later.
