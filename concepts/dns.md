# DNS — How It Works

## The Short Version

DNS (Domain Name System) is the internet's phonebook. It translates human-readable names like `google.com` into IP addresses like `142.250.80.46` that computers actually use to connect.

## How a DNS Lookup Works

When you type `google.com` in your browser:

```
1. Your computer asks your configured DNS server: "what's the IP for google.com?"
2. If the DNS server knows (it's cached), it answers immediately.
3. If not, it asks the root DNS servers → then the .com servers → then Google's servers.
4. The answer comes back: 142.250.80.46
5. Your browser connects to that IP.
```

This whole process takes milliseconds and happens for every domain you visit.

## What Pi-hole Does

Pi-hole sits between your devices and the internet as a DNS server. When your device asks "what's the IP for ads.doubleclick.net?", Pi-hole checks its blocklist and returns nothing (or `0.0.0.0`) instead of the real IP. The ad never loads because your browser can't find the server.

## Key Terms

**Resolver** — A DNS server that takes queries from clients and goes out to find the answer. Pi-hole is a resolver.

**Open Resolver** — A resolver that accepts queries from any IP on the internet, not just trusted clients. This is a security problem. See [the incident doc](../incidents/2026-06-20-open-dns-resolver.md).

**Recursive query** — When a resolver doesn't know the answer and goes out to find it by asking other DNS servers up the chain.

**TTL (Time to Live)** — How long a DNS answer can be cached before it needs to be looked up again.

**Port 53** — The standard port DNS runs on, both TCP and UDP. UDP is used for most queries (faster). TCP is used for large responses or zone transfers.

## Why Port 53 Open to the Internet Is Dangerous

DNS responses are often much larger than DNS requests. A 50-byte query can generate a 3000-byte response. Attackers exploit this asymmetry — they spoof a victim's IP, send your server small queries, and your server floods the victim with large responses. This is DNS amplification. See [the incident](../incidents/2026-06-20-open-dns-resolver.md).
