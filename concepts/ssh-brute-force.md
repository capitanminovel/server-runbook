# SSH Brute Force — What It Is and How to Defend Against It

## What's happening

SSH (port 22) is exposed to the internet so you can log in remotely. Bots scan for open port 22 and then hammer it with username/password combinations — "root/root", "admin/admin", "ubuntu/ubuntu123", thousands of combinations, over and over. This is SSH brute force. It's happening to every server with port 22 open, constantly.

On CapitanMinovel: 1,442 failed SSH auth attempts in a single 24-hour period. One IP (`92.182.32.104`) sent 873 of them alone.

This is not targeted at you. Bots just scan all IPs, find open SSH, and start guessing. It costs the attacker almost nothing to run.

## Why it usually doesn't work on this server

**Key-based auth only.** SSH is configured to require an SSH key, not a password. There's no password to guess. A brute force attack against key-based auth accomplishes nothing — the bot can try a million passwords and none of them will work because passwords aren't accepted.

This is the single most important SSH hardening measure. Everything else is secondary.

## How to check your SSH auth config

```bash
grep -E 'PasswordAuthentication|PermitRootLogin|PubkeyAuthentication' /etc/ssh/sshd_config
```

You want to see:
```
PasswordAuthentication no
PubkeyAuthentication yes
```

If `PasswordAuthentication` is `yes`, brute force attacks are a real threat. Change it to `no` and restart sshd.

## Detecting brute force in logs

```bash
# Count total auth failures in last 24h
journalctl --since "24 hours ago" -u ssh | grep -c "Failed password\|Invalid user"

# See which IPs are attacking and how many times
journalctl --since "24 hours ago" -u ssh | grep -oP 'from \K[\d\.]+' | sort | uniq -c | sort -rn | head -10

# See what usernames they're trying
journalctl --since "24 hours ago" -u ssh | grep -oP 'user \K\S+|Invalid user \K\S+' | sort | uniq -c | sort -rn | head -10
```

The usernames attackers try reveal what they're looking for: `root`, `admin`, `ubuntu`, `pi`, `git`, `deploy`. If you see a username you actually use on the list, that's notable.

## Defense layers on this server

**Layer 1 — Key-based auth (most important)**
No password = nothing to brute force. Even if all other defenses failed, this stops the attack.

**Layer 2 — fail2ban sshd jail**
Bans IPs after 3 failed attempts for 48 hours. Since key-based auth is enabled, "failed attempts" here means the bot connected and tried something that was rejected (wrong key, wrong user, etc.). fail2ban catches and bans the source IP quickly.

**Layer 3 — Manual UFW bans for known heavy hitters**
When a single IP sends hundreds of attempts before fail2ban catches it (or before fail2ban was installed), manually add a UFW deny rule:
```bash
ufw deny from 92.182.32.104 to any comment "ssh-bruteforce-manual-ban"
```

**Layer 4 (optional) — `ufw limit ssh`**
Rate-limits SSH connections at the firewall level: drops connections from any IP that attempts more than 6 connections in 30 seconds. Acts before fail2ban even sees anything.
```bash
ufw limit ssh comment "SSH rate limit"
```
Note: this replaces a plain `ufw allow ssh` rule, don't have both.

## What you don't need (and why)

**Moving SSH to a non-standard port** — commonly suggested, genuinely reduces noise (bots scan port 22 by default). But it's security through obscurity, not real security, and it adds operational friction. Not worth it if you have key-based auth + fail2ban.

**Port knocking** — requires sending packets to specific ports in sequence before SSH opens. Very annoying operationally. Not worth it.

**Allowlisting your home IP for SSH** — would eliminate all brute force noise. But if your home IP changes (most residential ISPs rotate IPs), you lock yourself out. Risky on a server you manage remotely.

## Commands to check current status

```bash
# See what IPs fail2ban has currently banned for SSH
fail2ban-client status sshd

# See manual UFW bans
ufw status | grep bruteforce

# Watch SSH auth log in real time (useful if you suspect an active attack)
journalctl -u ssh -f

# Verify password auth is disabled
sshd -T | grep passwordauthentication
```
