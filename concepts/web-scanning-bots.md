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
