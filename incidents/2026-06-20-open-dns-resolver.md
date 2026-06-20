# 2026-06-20 — Open DNS Resolver / Bandwidth Spike

## What Happened

Noticed an unexplained bandwidth spike over the preceding ~6 hours on the DigitalOcean droplet.

## Root Cause

Pi-hole was configured to listen on `0.0.0.0:53` (all interfaces), and UFW had port 53 open to `Anywhere`. This made the server a **public open DNS resolver** — anyone on the internet could send it DNS queries and it would answer them.

Confirmed in `/var/log/pihole/pihole.log`:
```
query . from 204.76.203.15        ← external IP querying root DNS
query[TXT] version.bind from 66.132.224.80   ← censys-scanner.com probing
```

## Why This Causes a Bandwidth Spike

**Free rider abuse:** Bots scan the internet for open resolvers and route their DNS traffic through them. Your server does the work and uses your bandwidth.

**DNS Amplification:** An attacker spoofs a victim's IP and sends your server small DNS queries (~50 bytes). Your server sends large responses (~3000 bytes) back to the victim. Multiplied across millions of requests, the victim gets flooded and your bandwidth spikes. Your server becomes an unwitting weapon in a DDoS attack.

## Fix

Removed open UFW rules and replaced with trusted-IP-only rules:

```bash
# Remove open rules
ufw delete allow 53/tcp
ufw delete allow 53/udp
ufw delete allow 8080/tcp
ufw delete allow 8443/tcp

# Allow only from trusted IP
ufw allow from <YOUR_IP> to any port 53 proto tcp comment "Pi-hole DNS TCP - trusted"
ufw allow from <YOUR_IP> to any port 53 proto udp comment "Pi-hole DNS UDP - trusted"
ufw allow from <YOUR_IP> to any port 8080 proto tcp comment "Pi-hole Admin HTTP - trusted"
ufw allow from <YOUR_IP> to any port 8443 proto tcp comment "Pi-hole Admin HTTPS - trusted"
```

Verify with:
```bash
ufw status verbose
```

## What I Learned

- Pi-hole is meant for a private network. On a public VPS it needs to be locked down explicitly.
- UFW `deny (incoming)` default only blocks ports with no rule — explicit `allow` rules override it for anyone.
- Open resolvers are actively scanned for and exploited within hours of being exposed.
- DNS amplification doesn't require the attacker to "break in" — your server just follows normal DNS protocol, which is why it's hard to detect.

## Follow-Up

- [ ] Configure Pi-hole to only bind to `localhost` rather than `0.0.0.0` (deeper fix beyond just firewall)
- [ ] If home IP is dynamic, update UFW rule when IP changes, or set up WireGuard VPN as a permanent solution
- [ ] See [DNS concept doc](../concepts/dns.md) for how DNS works
- [ ] See [Firewalls concept doc](../concepts/firewalls-ufw.md) for how UFW rules work
