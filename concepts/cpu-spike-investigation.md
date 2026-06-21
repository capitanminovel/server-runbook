# CPU Spike Investigation

## What this is
How to figure out what caused a CPU spike on the server — what commands to run, what's normal, and what's worth worrying about.

## How to investigate

Run these in parallel first:

```bash
# 1. System journal — errors/warnings in the last 6 hours
journalctl --since "6 hours ago" -p warning..err --no-pager | tail -100

# 2. Cron activity — what scheduled jobs ran and when
journalctl --since "6 hours ago" -u cron --no-pager

# 3. Who logged in
last -n 20 && who
```

Then drill into specific services:

```bash
# Check apt and firmware update services (common spike sources)
journalctl --since "6 hours ago" -u apt-daily.service -u apt-daily-upgrade.service -u fwupd-refresh.service --no-pager

# Check what's in cron.hourly
ls -la /etc/cron.hourly/
cat /etc/cron.hourly/*
```

Key thing to look for: when systemd finishes a service it logs:
```
service.service: Consumed 21.578s CPU time.
```
That line is a definitive answer — no guessing required.

To catch a spike in the act:
```bash
ps aux --sort=-%cpu | head -15
top -b -n 1 | head -20
systemd-cgtop -n 1
```

## What's normal (you'll see these regularly)

| What | Why | Notes |
|---|---|---|
| `apt-daily.service` | Downloads package index (like `apt-get update`) | Short CPU spike, harmless |
| `apt-daily-upgrade.service` | Actually installs packages | Longer spike, also normal |
| `fwupd-refresh` | Firmware update check | Finishes in milliseconds |
| `debian-sa1` / `sysstat` | System metrics collector | Runs every 10 min, trivial CPU |
| `cron.hourly droplet-agent` | DigitalOcean agent update check | Has a built-in random sleep so sessions look long — not actually doing much |

## What's worth investigating

- A cron session open for a long time with no sleep/wait explanation
- `apt-daily-upgrade.service` taking unusually long (could mean many packages installing)
- Any process running from `/tmp/`, `/dev/shm/`, or `/var/tmp/` — common malware drop locations
- An unfamiliar process name in `ps aux` during a spike
- A service that started but never finished (hung process) — shows up as open session with no close in journal
- High CPU from `pihole-FTL` — could indicate a DNS amplification attempt or a domain being hammered

## What I learned
- `systemd` logs CPU time consumed at the end of every service run — use that as ground truth
- `journalctl -u cron` shows every cron job with open/close times, so you can spot long-running jobs
- The DropletAgent hourly cron has a randomized sleep baked in — long session time is expected
- UFW BLOCK lines in the journal are normal internet background noise (port scanners), not a sign of a problem
