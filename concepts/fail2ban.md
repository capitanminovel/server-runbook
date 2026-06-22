# fail2ban — What It Is and How It Works

## What it is

fail2ban watches your log files and automatically bans IPs that show attack patterns — too many failed SSH logins, too many probe requests to your web server, etc. When it bans an IP, it adds a firewall rule to block all traffic from that address for a set amount of time.

It's not a replacement for UFW. UFW is your static ruleset (what's always open/closed). fail2ban is a dynamic layer on top — it reacts to what's actually happening.

## The mental model

```
logs → fail2ban reads them → counts bad behavior per IP → hits threshold → writes firewall rule → removes rule after bantime
```

That's the whole thing. There's no magic — it's a log watcher that writes and removes firewall rules.

## Key concepts

**Jail** — a combination of a log source, a filter (regex), and an action. Each jail watches one service. You can have a jail for SSH, one for nginx probes, one for nginx bots, etc.

**Filter** — a regex pattern that defines what "bad behavior" looks like in the log. Stored in `/etc/fail2ban/filter.d/`. fail2ban ships with many pre-built filters for common services.

**findtime** — the window of time fail2ban looks back when counting failures. e.g. `findtime = 5m` means "count failures in the last 5 minutes."

**maxretry** — how many matches before the ban fires. `maxretry = 3` means ban after 3 matches within findtime.

**bantime** — how long the ban lasts. `bantime = 24h`. After this, the rule is removed automatically.

**banaction** — what actually does the banning. On this server: `banaction = ufw`, which means fail2ban runs `ufw insert 1 deny from <IP>` to ban and `ufw delete deny from <IP>` to unban.

## Config file layout

Never touch `/etc/fail2ban/jail.conf` — it's owned by the package and gets overwritten on upgrades.

Your config goes in `/etc/fail2ban/jail.local`. Anything in `jail.local` overrides `jail.conf`. This is the standard Linux override pattern (same idea as nginx's sites-available, or systemd drop-ins).

```
/etc/fail2ban/
├── jail.conf          ← package-owned, don't touch
├── jail.local         ← your config (overrides jail.conf)
└── filter.d/
    ├── sshd.conf      ← built-in SSH filter
    ├── nginx-botsearch.conf   ← built-in nginx bot filter
    └── nginx-probe.conf       ← custom filter we wrote
```

## This server's configuration (`/etc/fail2ban/jail.local`)

```ini
[DEFAULT]
bantime  = 24h
findtime = 10m
maxretry = 5
banaction = ufw

[sshd]
enabled  = true
port     = ssh
logpath  = %(sshd_log)s
backend  = %(sshd_backend)s
maxretry = 3
bantime  = 48h

[nginx-probe]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/access.log
filter   = nginx-probe
maxretry = 8
findtime = 5m
bantime  = 24h

[nginx-botsearch]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/access.log
filter   = nginx-botsearch
maxretry = 2
findtime = 1m
bantime  = 24h
```

The custom filter (`nginx-probe`) is at `/etc/fail2ban/filter.d/nginx-probe.conf`. It catches requests for `.php` files, `.env` paths, `.git` paths, `wp-admin`, and path traversal patterns.

## Key commands

```bash
# Check all jails are loaded and running
fail2ban-client status

# See details for a specific jail — current bans, total bans, filter matches
fail2ban-client status sshd
fail2ban-client status nginx-probe

# Manually ban an IP (useful when you spot an attacker before fail2ban does)
fail2ban-client set sshd banip 1.2.3.4

# Manually unban an IP (in case of false positive)
fail2ban-client set sshd unbanip 1.2.3.4

# Test a filter against a log file — shows you what it would match
fail2ban-regex /var/log/nginx/access.log /etc/fail2ban/filter.d/nginx-probe.conf

# Watch fail2ban's own log in real time
tail -f /var/log/fail2ban.log

# Restart after config changes
systemctl restart fail2ban

# Test config before restarting
fail2ban-client -t
```

## What fail2ban does NOT do

- It does not block the first N attempts (up to `maxretry`). A threshold of 3 means the attacker gets 3 free tries.
- It does not protect against distributed attacks where each IP only tries once. The `.env` scanner from `179.43.168.58` sent 126 requests from one IP — caught. If those 126 requests came from 126 different IPs, fail2ban would miss it.
- It does not alert you in real time. It bans silently. Check `fail2ban-client status` or the daily attack summary to see what's been blocked.
- If fail2ban goes down, the existing ban rules stay in UFW (they were written there). New offenders won't be banned until it's back up.

## How to know if a jail is actually working

```bash
fail2ban-client status nginx-probe
```

Look at "Total failed" — if it's 0 after traffic has been flowing, the filter regex isn't matching the log or the log path is wrong. Use `fail2ban-regex` to test the filter against the actual log file.

## Why `banaction = ufw` is the right choice here

The default banaction uses `iptables` directly. We use `ufw` so all firewall rules stay in one place and `ufw status` gives a complete picture of what's blocked. If you used the default iptables action, fail2ban bans would be invisible in `ufw status`.
