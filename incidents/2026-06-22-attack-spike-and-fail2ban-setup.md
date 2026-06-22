# 2026-06-22 — Attack Spike Investigation + fail2ban Setup

## What happened
Investigated bandwidth/CPU/disk spike on CapitanMinovel. Found three distinct attack waves in nginx logs and a heavy SSH brute-force campaign. Also discovered a `.git` directory in the nginx webroot that could have leaked Pi-hole admin UI source metadata.

## What was actually spiking

**CPU/Disk**: Not spiking. sar history showed ~98% idle all day. The apparent spike was Claude Code running during the investigation.

**Bandwidth (nginx)** — 3 attack waves:
| Time | Requests | What it was |
|------|----------|-------------|
| 01:00 | 168 | `.env` credential harvester from `179.43.168.58` — tried 126 `.env` path variants |
| 05:00 | 201 | General web probe |
| 14:00 | 284 | PHP webshell + WordPress scanner via Cloudflare-proxied IPs — probing for uploaded shells, wp-admin, `.git/config`, path traversal |

**Port scan**: 4,456 UFW blocks in 24h. Top offender: `34.174.70.211` (GCP IP, 256 blocks) scanning Redis, Docker API, Elasticsearch, RDP, MySQL, etc.

**SSH brute force**: 1,442 auth failures in 24h. Top attacker: `92.182.32.104` (873 attempts alone).

All attacks were blocked. Nothing got through.

## Fixes applied

### 1. nginx .git/.env block
Added to `/etc/nginx/sites-available/default`:
```nginx
location ~ /\.(git|env|htaccess) {
    deny all;
    return 404;
}
```
This prevents serving Pi-hole's `.git/config` from `/var/www/html/admin/.git/`.

### 2. fail2ban installed and configured
`/etc/fail2ban/jail.local`:
- **sshd** — ban after 3 failures, 48h ban
- **nginx-probe** — custom filter catching PHP webshell/wp-admin/env/git probes, ban after 8 hits in 5min, 24h ban
- **nginx-botsearch** — ban after 2 hits in 1min, 24h ban

Custom filter at `/etc/fail2ban/filter.d/nginx-probe.conf`.

`banaction = ufw` so fail2ban writes UFW rules directly.

### 3. Manual SSH brute-force bans
Top 5 SSH brute-force IPs manually banned via UFW (tagged `ssh-bruteforce-manual-ban`):
- `92.182.32.104` (873 hits)
- `176.65.139.181` (145 hits)
- `162.243.8.173` (76 hits)
- `222.108.100.117` (53 hits)
- `43.153.59.240` (52 hits)

### 4. Daily attack summary script
`/usr/local/bin/attack-summary` — runs at 07:00 daily via cron.
Read output with: `journalctl -t attack-summary --since today`

## Why the attacks happen
This is standard internet background noise for any public IP. Bots constantly scan all IPv4 space looking for:
- Open SSH with weak passwords (most common)
- WordPress installs to inject shells
- Exposed `.env` files with API keys/DB creds
- Open Docker APIs (2375), Redis (6379), Elasticsearch (9200) with no auth

UFW's default-deny stops port scans. fail2ban adds active banning for repeat offenders. nginx returning 404 stops web probes.

## What I learned

### Internet background noise is constant and automated
Any public IP gets scanned within minutes of coming online. These aren't humans — they're bots running 24/7 against every IPv4 address. The `.env` scanner at 01:00 tried 126 paths in sequence; that's a script, not a person. The PHP webshell scanner at 14:00 had a list of hundreds of filenames it tried one by one. The SSH attacker at 92.182.32.104 sent 873 attempts — bots just hammer until banned or successful. This is background radiation on the internet. It never stops.

### UFW blocks don't mean silence — they mean your server saw the packet and dropped it
When UFW blocks something, the packet still arrived at your network interface. The kernel received it, UFW inspected it, and dropped it. You still pay a tiny bit of CPU and bandwidth per blocked packet. 4,456 UFW blocks in 24h is trivial — but if a scan were 10x more aggressive, you'd see iowait and CPU tick up from kernel processing alone. This is why blocking at the source (ISP/DigitalOcean firewall) is better than UFW if volume becomes a problem.

### fail2ban works by watching logs and writing firewall rules — it's not magic
fail2ban reads log files (nginx access log, sshd journal) on a timer, runs regex filters against new lines, counts matches per IP, and when an IP hits `maxretry` within `findtime`, it adds a ban rule. With `banaction = ufw`, it literally runs `ufw insert 1 deny from <IP>`. When `bantime` expires, it removes the rule. That's the whole mechanism. Understanding this matters because:
- fail2ban only catches things *after* they show up in a log — it can't block the first N attempts (up to maxretry)
- If a log format changes (nginx config update, different log path), the jail silently stops working
- `fail2ban-client status <jail>` is how you verify it's actually reading the right logs

### The `jail.local` / `jail.conf` split exists for package safety
`/etc/fail2ban/jail.conf` is owned by the package. When fail2ban upgrades, that file gets overwritten. `jail.local` is your file — it overrides anything in `jail.conf` and never gets touched by apt. Same pattern applies to nginx (`/etc/nginx/nginx.conf` vs your site files in `sites-available/`) and many other services. The principle: **your config goes in the override file, never in the package-owned file**.

### A `.git` directory in a webroot is a repo exfiltration risk
When you clone a git repo into a web-served directory, the `.git/` folder contains the full object database — meaning someone can reconstruct your entire source code by requesting git object files one by one. There are automated tools that do exactly this. The risk here (Pi-hole admin UI source) was low since it's public code anyway, but if it were *your* app source with hardcoded credentials or business logic, it would be a serious leak. Rule: never clone into a webroot. If you must, block `/.git` in nginx.

### Why PHP webshell scans hit a FastAPI server
The scanners don't know what tech stack you're running. They probe everything. A PHP webshell is a file like `shell.php` that an attacker previously uploaded (through a file upload vulnerability) and can now use to run commands on the server. The scan is checking: "did anyone leave a shell here?" nginx returning 404 means the file doesn't exist — exactly right. This is why nginx's default `try_files $uri =404` behavior is your friend.

### Cloudflare IPs in nginx logs don't mean Cloudflare is attacking you
Cloudflare runs a massive CDN. Attackers can route their traffic through Cloudflare's network (by putting a Cloudflare-proxied domain in front of their scanner). The IP in your nginx log is the Cloudflare edge node, not the actual attacker. This is why you can't just block `104.23.x.x` — it would block legitimate Cloudflare traffic too. fail2ban handles this correctly by banning per-IP after enough bad requests, regardless of whether it's a real attacker or a proxied one.

### `sar` vs `top` — snapshot vs history
`top` and `ps aux` show you what's happening *right now*. `sar` (System Activity Reporter) stores samples every 10 minutes in `/var/log/sysstat/saXX` (XX = day of month). When investigating a spike that already happened, `sar` is the only tool that has historical data. `sar -u` shows CPU, `sar -n DEV` shows network, `sar -d` shows disk I/O. If sysstat isn't installed, you have no historical record — that's why it matters to have it running.

## Commands for ongoing monitoring
```bash
# Daily summary
/usr/local/bin/attack-summary
journalctl -t attack-summary --since today

# fail2ban jail status
fail2ban-client status sshd
fail2ban-client status nginx-probe
fail2ban-client status nginx-botsearch

# See all currently banned IPs
fail2ban-client status sshd | grep "Banned IP"

# Top UFW blockers right now
journalctl --since "24 hours ago" -k | grep "UFW BLOCK" | grep -oP 'SRC=\S+' | sort | uniq -c | sort -rn | head -10

# nginx probe count
grep -cE '\.(php|env|git)|wp-admin|wp-login' /var/log/nginx/access.log
```

## Follow-up
- Consider rate-limiting SSH to port 22 with `ufw limit ssh` for an extra layer (drops connections after 6 in 30s)
- The manual UFW bans (`ssh-bruteforce-manual-ban`) are persistent but not time-limited — fail2ban will handle future offenders automatically
- Monitor fail2ban logs for a few days to tune thresholds if getting false positives
