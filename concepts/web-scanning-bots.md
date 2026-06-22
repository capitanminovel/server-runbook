# Web Scanning Bots — What They Are and When to Care

## What's Happening

Every public IP on the internet gets probed constantly by automated bots. These bots scan the entire IPv4 address space (all ~4 billion IPs) looking for known vulnerabilities. This is not targeted at you — it's a numbers game. They try every IP, and act on the ones that respond with something vulnerable.

## Common Things They Probe For

| Path | What They're Looking For |
|---|---|
| `/.env` | Credentials file accidentally left in web root — DB passwords, API keys |
| `/.git/config` | Git repo exposed via web — can download your entire source code |
| `/wp-admin/install.php` | Unprotected WordPress admin install |
| `/phpinfo.php` | PHP info page — reveals server config, paths, versions |
| `/eval-stdin.php` | Known PHP RCE (remote code execution) vulnerability |
| `/cgi-bin/../../bin/sh` | Path traversal → shell execution |
| `/admin/config.php` | Generic admin panel guessing |
| `mstshash=zgrab` | zgrab scanner — open source internet scanning tool |

## When to Ignore Them

If all of these return **404** on your server, you're fine. The bots move on. They're not going to brute-force a path that doesn't exist.

## When to Actually Worry

- A `.env` file is inside a directory nginx serves statically
- Your app has a `/phpinfo` or `/debug` endpoint that reveals server internals
- You're running WordPress or another CMS with known unpatched CVEs
- An endpoint accepts a file path as user input without validation (path traversal)
- An endpoint takes user input and puts it directly in a database query (SQL injection)

## How to Tell What nginx Is Serving

Check your nginx config to see what's mapped to what:

```bash
cat /etc/nginx/sites-enabled/*
```

Any `root` directive points to a directory being served. Make sure:
- `.env` files are not inside those directories
- `.git` folders are not inside those directories
- No debug endpoints are accessible in production

## Why Cloudflare IPs Show Up in Attack Logs

Sometimes nginx access logs show attack traffic coming from Cloudflare IP ranges (`104.23.x.x`, `172.70.x.x`, `162.158.x.x`). This doesn't mean Cloudflare is attacking you.

Attackers can route their scanner traffic through Cloudflare's network by putting a Cloudflare-proxied domain in front of their tool. The actual scanner is somewhere else; the Cloudflare edge node is just a relay. Your nginx log sees the edge node IP, not the attacker's real IP.

**What to do:** Don't block the Cloudflare IP range — that would break legitimate Cloudflare-proxied traffic. fail2ban handles this correctly by banning the specific IP after enough bad requests, regardless of whether it's a direct attacker or a proxy.

## The `.git` Directory Risk

If you clone a git repo into a directory that nginx serves as static files, the `.git/` folder is accessible over HTTP. This is a real problem.

Git stores the full history and content of a repo as object files under `.git/objects/`. An attacker can request these files one by one and reconstruct your entire source code — including any hardcoded secrets, API keys, or credentials that were ever committed, even if they were removed later.

There are automated tools built specifically to do this (e.g. `git-dumper`).

**How to block it in nginx:**
```nginx
location ~ /\.git {
    deny all;
    return 404;
}
```

**The broader rule:** any file or directory starting with `.` in a webroot is probably dangerous to expose. The nginx block `location ~ /\.(git|env|htaccess)` covers the most common cases.

On this server: Pi-hole's admin UI source lives at `/var/www/html/admin/` and includes a `.git` directory. This block is in the nginx default server config.

## Useful Tools for Monitoring

**fail2ban** — Watches logs and auto-bans IPs that repeatedly hit 404s or fail SSH logins. Reduces noise significantly.

```bash
apt install fail2ban
# Default config handles SSH; nginx jail needs a bit of setup
```

**GoAccess** — Terminal-based nginx log analyzer. Good for seeing traffic patterns at a glance.

```bash
apt install goaccess
goaccess /var/log/nginx/access.log --log-format=COMBINED
```
