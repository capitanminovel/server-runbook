# SSRF (server-side request forgery)

## What it is
SSRF is when someone gets *your server* to make a web request on their behalf. The server sits in a
trusted position — it can reach things the public internet can't (its own localhost services, the
private network, the cloud provider's metadata address) — so a request it makes carries that trust.

Whenever a feature says "the server fetches a URL that a user typed or that came from user-supplied
data," it is a potential SSRF hole.

## Why it matters / how it works
Example: dispo_menu's Strain Generator fetches pages from an admin-entered supplier website. Without
protection, a URL like `http://127.0.0.1:5432/` (the database), `http://10.0.0.5/` (private network) or
`http://169.254.169.254/latest/meta-data/` (the metadata service, which on some clouds hands out
credentials) would make our server connect to those and, depending on what comes back, leak it.

Two subtler ways in: a **redirect** (a public site answers `302 → http://127.0.0.1/...`), and a
**hostname that resolves to a private IP** (`evil.example.com → 127.0.0.1`). Checking only the text of
the URL misses both — you have to check where the name *resolves* and re-check on every redirect hop.

## How it's set up on this server
`apps/api/app/ai/page_fetcher.py` (dispo_menu) is the only place the API fetches arbitrary URLs:
- only `http`/`https`, only ports 80/443, no `user:pass@` in the URL
- hostname is resolved and **every** address must be globally routable (`ipaddress.is_global`) —
  loopback, private ranges, link-local (169.254.x.x) and reserved ranges are refused
- redirects are followed by hand (max 4), re-validating each hop
- response size and time are capped; robots.txt is honoured
- tested against 127.0.0.1, 169.254.169.254, localhost, 10.0.0.1, `file://`, odd ports, embedded creds —
  all refused

## What it does NOT do
- **DNS rebinding:** DNS could answer differently between our check and the actual connection. Closing
  that means pinning the connection to the IP we validated. Accepted for an authenticated, admin-only
  feature; revisit before exposing URL fetching to untrusted users.
- It doesn't make the *content* safe — fetched pages are untrusted text. They're handed to the model as
  labeled research material, never as instructions.
- It isn't a firewall: it protects this one code path only.

## Key commands
```bash
# the guard, exercised directly (from /root/Dispo_menu/apps/api)
venv/bin/python -c "from app.ai import page_fetcher as pf; pf._validate_url('http://169.254.169.254/')"
# -> FetchError: ... resolves to a non-public address; refusing to fetch
```
