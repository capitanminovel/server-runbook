# Swap Space Setup

## What happened
Server had zero swap, causing memory pressure crashes when RAM (961MB total) filled up (e.g., running Claude Code sessions + two FastAPI apps simultaneously).

## Why
Without swap, when RAM is exhausted the kernel starts force-killing processes (OOM killer). Adding a swap file gives the OS a disk-backed overflow buffer. It's slower than RAM but prevents crashes.

## Fix — 2GB swap file

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
```

Persisted in `/etc/fstab`:
```
/swapfile none swap sw 0 0
```

Swappiness set to 10 (use swap as last resort, not aggressively):
```bash
echo "vm.swappiness=10" >> /etc/sysctl.conf
sysctl vm.swappiness=10
```

## What I learned
- `fallocate` creates a pre-allocated file (faster than `dd`)
- `chmod 600` is required — swap files must not be world-readable
- `mkswap` formats it, `swapon` activates it
- `/etc/fstab` entry makes it survive reboots
- `swappiness=10` means: only use swap when RAM is 90%+ full, preserving RAM for active processes

## Verify
```bash
free -h          # should show 2GB swap
swapon --show    # shows active swap files
```
