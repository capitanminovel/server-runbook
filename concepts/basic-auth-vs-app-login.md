# Two logins: nginx Basic Auth vs. the app's own login

## What it is
The dev site (`dispo-admin.dev.withcapitan.com`) asks for a password twice, and they are unrelated:
1. **nginx Basic Auth** — the browser's grey pop-up. One shared username/password for the whole site.
2. **The app login** — the "Welcome to the Legit Cannabis Dispo Tool" page, per person (email + password).

## Why it matters / how it works
- **Basic Auth is a gate at the door.** nginx checks it *before* any of our code runs. Its only job on
  the dev site is keeping the public internet — scanners, bots, curious visitors — away from an
  unfinished app. It knows nothing about who you are; everyone shares one credential.
- **The app login is identity.** It sets a session cookie tied to a staff user, and that user's
  `dispensary_id` decides which tenant's data every request can touch. This is what keeps one
  dispensary's data separate from another's in the shared database. Basic Auth can't do that.

They are different layers solving different problems (keep strangers out vs. know who you are).

## How it's set up on this server
- Basic Auth: in the nginx site config for the dev hostnames (htpasswd file). Dev-only.
- App login: `apps/api/app/routes/auth.py` (`/api/auth/login`), Starlette `SessionMiddleware` cookie,
  bcrypt password hashes, rate-limited by nginx (`dispo_login` zone, 10 requests/min).
- Production plan: the Basic Auth layer goes away; the app login is the real security boundary.

## What it does NOT do
- Basic Auth does not protect data between tenants, and is not a substitute for the app login.
- Neither replaces per-request tenant checks in the API (`dispensary_id` re-checked on every FK lookup).

## Key commands
```bash
ls /etc/nginx/sites-enabled/                 # find the dev site configs
nginx -t && systemctl reload nginx           # after editing them
```
