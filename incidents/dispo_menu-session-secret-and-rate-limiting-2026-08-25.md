# dispo_menu — Weak Session Secret + No Rate Limiting on AI Endpoint

**Date:** 2026-08-25

## What happened

User asked me to confirm the dispo_menu dev preview was secure and specifically flagged concern
about uncontrolled AI API usage. Rather than just reassuring them, I checked, and found two real
issues — neither had been exploited, both fixed same-session.

## Why

1. **`SESSION_SECRET` was still the placeholder value from the original foundation-scaffold
   session** (`local-dev-only-change-me-1f8a2c9d4e6b`, set 2026-08-02). This secret signs the
   Starlette session cookie. Anyone who obtained this exact string — plausible, since it's the
   kind of value that ends up in a shared doc, a screenshot, or committed by mistake elsewhere —
   could forge a valid signed session cookie for any `staff_user_id`, bypassing login entirely,
   without ever needing the actual password.
2. **No rate limiting existed on `/api/strains/` (the Strain Generator's POST endpoint, which
   calls the paid Claude API) or on `/api/auth/login`.** The only protection on strain generation
   was "must be logged in" — nothing capped how many times a valid session could trigger it. Since
   dispo_menu's `CLAUDE_API_KEY` is the same key used by `legit-buddy-private`
   (see `dispo_menu-strain-generator-2026-08-25.md`), unbounded calls here would burn the same
   budget as that production service. Login also had no lockout/backoff, so it was (weakly, given
   a 16-char random password) brute-forceable.

## Fix

1. Rotated `SESSION_SECRET` to a proper random value (`secrets.token_hex(32)`) in
   `/root/Dispo_menu/apps/api/.env`, restarted `dispo-menu-api`. This invalidates all existing
   sessions — expected, one-time inconvenience (everyone has to log in again).
2. Added nginx-level rate limiting — chose nginx over app-level middleware since it's a stronger
   layer (rejects before the request even reaches the app/database) and matches this server's
   existing pattern of doing access control at the nginx layer (Basic Auth, SSL, etc.):
   - New file `/etc/nginx/conf.d/dispo-menu-rate-limits.conf`:
     ```nginx
     limit_req_zone $binary_remote_addr zone=dispo_ai_generation:10m rate=30r/m;
     limit_req_zone $binary_remote_addr zone=dispo_login:10m rate=10r/m;
     ```
   - In `/etc/nginx/sites-available/dispo-api.dev.withcapitan.com`, added exact-match `location`
     blocks for `/api/strains/` (30r/m, burst 10, nodelay) and `/api/auth/login` (10r/m, burst 5,
     nodelay) ahead of the catch-all `location /`.
   - Rate applies to the whole `/api/strains/` path (GET list + POST generate both), since nginx
     can't cheaply split by HTTP method on one path without more complex config (`map` +
     `if`/`limit_except` gymnastics) — 30r/m is loose enough that normal admin use (occasional
     list refresh, occasional generate) never notices it, but caps a script/loop at a bounded
     rate instead of unlimited.
   - **Verified live, not just configured**: fired 20 rapid GET requests at `/api/strains/` —
     first 10 (the burst allowance) returned 200, the next 10 returned 503.

## What I learned

- A placeholder secret that looks obviously fake ("change-me") is exactly the kind of thing that
  survives past the setup session because it *works* — nothing breaks until someone actually
  exploits it. Worth a habit: grep `.env` files for anything containing "change" or "placeholder"
  before calling a deployment done, not just during initial setup.
- nginx's `limit_req` can't natively split by HTTP method on a single path — a real limitation
  worth knowing before reaching for it as a precision tool. It's fine as a coarse cap (this case),
  but a scenario needing "block POST but never GET" on the same path needs either `limit_except`
  combined with `if`, or handling it in the application layer instead.

## Follow-up

- [ ] No app-level rate limiting exists yet (nginx-level is the only layer) — if this ever moves
      behind a different proxy or the nginx config gets refactored, the protection could silently
      disappear. Worth adding a defense-in-depth app-level limit too, not just relying on nginx.
- [ ] Still only one staff account, one shared Basic Auth credential, no MFA, no monitoring/alerting
      on unusual usage patterns — acceptable for a dev preview, not for real customer data.
- [ ] Anthropic API key is still shared with legit-buddy-private — a real per-service key would
      let usage/cost be attributed and capped independently.
