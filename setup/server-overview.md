# Server Overview

## Provider & Specs

- **Provider:** DigitalOcean (Droplet)
- **OS:** Ubuntu 22.04+
- **Public IP:** 167.99.235.92
- **Hostname:** CapitanMinovel

## Services

### nginx
Reverse proxy and web server. Listens on ports 80 and 443. Routes traffic to internal apps.

Internal apps are bound to `127.0.0.1` only — they're never directly reachable from the internet. nginx is the only thing that talks to them.

### Pi-hole (FTL)
DNS server and ad blocker. Runs as the `pihole` user.

- Binds to `0.0.0.0:53` (should be locked down to localhost if not serving a local network)
- Admin panel on ports 8080 / 8443
- UFW rules restrict DNS and admin access to trusted IP only (fixed June 2026)

### money_buddy
Python FastAPI app. Runs via uvicorn on `127.0.0.1:8000`. Managed by systemd.

```bash
systemctl status money_buddy
journalctl -u money_buddy -f
```

### capitans_terps
Python FastAPI app. Runs via uvicorn on `127.0.0.1:8001`. Managed by systemd.

```bash
systemctl status capitans_terps
journalctl -u capitans_terps -f
```

### SSH
Standard OpenSSH on port 22. Key-based auth (DigitalOcean default). Open to all IPs.

## Useful Locations

| Thing | Path |
|---|---|
| nginx config | `/etc/nginx/sites-enabled/` |
| nginx access log | `/var/log/nginx/access.log` |
| nginx error log | `/var/log/nginx/error.log` |
| Pi-hole log | `/var/log/pihole/pihole.log` |
| money_buddy app | `/opt/money_buddy/` |
| capitans_terps app | `/opt/capitans_terps/` |
| UFW rules | `ufw status verbose` |
| System logs | `journalctl -f` |

## Quick Health Check Commands

```bash
systemctl status nginx
systemctl status pihole-FTL
ufw status verbose
ss -tlnp          # what's listening on which ports
df -h             # disk usage
free -h           # memory usage
```
