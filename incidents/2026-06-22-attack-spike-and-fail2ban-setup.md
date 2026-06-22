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
- `sar` is the tool for historical CPU data (10-min samples in `/var/log/sysstat/`)
- fail2ban with `banaction = ufw` is the right combo — it manages UFW rules automatically
- Never edit `jail.conf` directly — always use `jail.local` (survives package updates)
- Read fail2ban output with `journalctl -t attack-summary` or `fail2ban-client status <jail>`
- Pi-hole admin UI source lives at `/var/www/html/admin/` with a `.git` dir — always block `.git` in nginx webroot

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
