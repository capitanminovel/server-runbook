# Firewalls and UFW

## What a Firewall Does

A firewall controls which network traffic is allowed to reach your server. Think of your server as a building with 65,535 doors (ports). Most are locked. The firewall decides which doors are open, and optionally who is allowed to knock.

## UFW — Uncomplicated Firewall

UFW is the firewall tool on Ubuntu/Debian. It's a simplified interface over `iptables`, which is the actual Linux kernel firewall.

### Key Commands

```bash
ufw status verbose          # see all current rules
ufw allow 80/tcp            # open port 80 to anyone
ufw allow from 1.2.3.4 to any port 53 proto udp   # open port 53 only to one IP
ufw delete allow 53/udp     # remove a rule
ufw enable / ufw disable    # turn firewall on/off
```

### How Rules Are Evaluated

UFW has a **default policy** and then explicit rules. On this server:
- Default incoming: **deny** — anything not explicitly allowed is blocked
- Default outgoing: **allow** — the server can initiate connections out freely

Rules are processed and the first match wins. So if you have:
```
allow from 1.2.3.4 to any port 53
deny 53
```
Traffic from `1.2.3.4` hits the first rule and is allowed. Everything else hits the deny rule and is blocked.

### The `0.0.0.0` vs Specific IP Difference

```bash
ufw allow 53/udp                              # allows from 0.0.0.0 = ANYONE
ufw allow from 1.2.3.4 to any port 53/udp    # allows only from that IP
```

This is the difference between an open service and a locked-down one. Always be explicit about *who* should be able to reach a service, not just *what port* is open.

### Adding Comments to Rules

```bash
ufw allow from 1.2.3.4 to any port 53 proto udp comment "Pi-hole - home IP"
```

Comments show up in `ufw status` and make it obvious why a rule exists when you look at it months later.

## Ports Worth Knowing

| Port | Protocol | Service |
|---|---|---|
| 22 | TCP | SSH |
| 25 | TCP | SMTP (email) |
| 53 | TCP/UDP | DNS |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |

If you're not running a service on a port, it should be blocked. If you are running one, think hard about who actually needs to reach it.

## What This Server's Rules Protect Against

- Database ports (3306, 5432) are not open — if a database were misconfigured to listen on `0.0.0.0`, UFW would still block external access.
- Pi-hole DNS was open accidentally — fixed by restricting to trusted IP only.
- App ports (8000, 8001) are not exposed externally — nginx proxies to them internally only.
