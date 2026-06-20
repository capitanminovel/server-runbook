# Server Runbook

Personal knowledge base for DevOps, server security, and ops learnings.

Each entry follows the same structure: **what happened → why → fix → what I learned.**
Written right after incidents while details are fresh.

## Structure

```
/incidents      → things that went wrong and how they were resolved
/concepts       → explanations of tools, protocols, and ideas
/setup          → how the server and services are configured
```

## Server Overview

**Provider:** DigitalOcean  
**OS:** Ubuntu  
**IP:** 167.99.235.92  

### What's Running

| Service | Purpose | Port |
|---|---|---|
| nginx | Reverse proxy / web server | 80, 443 |
| Pi-hole (FTL) | DNS server + ad blocking | 53 (internal only) |
| money_buddy | Python/FastAPI app | 8000 (internal) |
| capitans_terps | Python/FastAPI app | 8001 (internal) |
| SSH | Remote access | 22 |

### Firewall Rules (UFW)

| Port | Open To | Reason |
|---|---|---|
| 22 | Anywhere | SSH access |
| 80 | Anywhere | nginx HTTP |
| 443 | Anywhere | nginx HTTPS |
| 53 | Trusted IP only | Pi-hole DNS — was open, fixed June 2026 |
| 8080/8443 | Trusted IP only | Pi-hole admin panel |

---

## Index

### Incidents
- [2026-06-20 — Open DNS Resolver / Bandwidth Spike](incidents/2026-06-20-open-dns-resolver.md)

### Concepts
- [DNS and How It Works](concepts/dns.md)
- [Firewalls and UFW](concepts/firewalls-ufw.md)
- [Web Scanning Bots — What They Are and When to Care](concepts/web-scanning-bots.md)
