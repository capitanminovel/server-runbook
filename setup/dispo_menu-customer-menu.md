# dispo_menu — customer menu + kiosk (v1, 2026-09-26)

Spec + decisions: `Dispo_menu/docs/customer-menu-spec.md`. Browse-only (no cart/ordering), for customers.

## Layout principle (decided 2026-09-26)
**Strain profile first, shelf last.** Stock comes and goes; the profile is what customers read. Cards: feel, taste,
terpenes, then a small "Flower 3.5 g · $55" line. Detail: about → feel → taste → terpenes → lineage → fact → reported
uses → "If you like this, try" → **On the shelf now** (sizes, prices). Kiosk showcase has no price. Also a
Daytime / Evening / Any time tag worked out from each strain's effects.

## Where it runs
- Web: https://dispo-menu.dev.withcapitan.com (phones/tablets/computers, 21+ gate) — `/strain/<id>` per strain.
- Kiosk: https://dispo-menu.dev.withcapitan.com/kiosk (landscape TV or tablet; no age gate; idle showcase after 60 s).
  On the device: Chrome in kiosk mode, e.g. `chrome --kiosk --noerrdialogs https://…/kiosk`.
- **Dev only:** both behind the same nginx password as the admin site + `noindex`, until the compliance review.
  To go public: remove the two `auth_basic` lines in `/etc/nginx/sites-available/dispo-menu.dev.withcapitan.com`.
- Static files in `/var/www/dispo-menu`, built by `./deploy.sh menu`.

## How the data flows (why it's cheap)
Refresh (5-min timer) → saves each linked product's sizes, prices, sale price, photo URL, THC and Indica/Sativa/Hybrid
→ `GET /api/public/menu/<subdomain>` builds the menu from our DB (never calls Sweed or the AI), cached 60 s in the app
→ nginx rate limit `dispo_public` (60/min per address, burst 30). A visitor or a kiosk costs one small JSON download.

## Security
- No login, so the endpoint returns an explicit **whitelist** of fields (`app/public_menu.py` `_item()`): no cautions,
  research sources, shop links, Sweed ids or staff data. `scripts/menu_check.py` fails if any other field appears.
- Menu site CSP: only our scripts/styles/fonts, images from `media-prime.sweedpos.com` (Sweed's photo server), data from
  our API; `frame-ancestors 'none'`. nginx gotcha: an `add_header` inside a `location` replaces ALL server-level
  `add_header`s — so the headers are repeated in `location /`.
- Photos are hot-linked from Sweed's image server (simplest). If Sweed changes URLs, the next refresh picks up the new ones.

## Hidden / switched
- Cautions: hidden. "Therapeutic" → "Reported uses" + not-medical-advice note; off switch `SHOW_REPORTED_USES`.
  Watch: values like "Depression", "Chronic pain", "Insomnia" are the likeliest health-claim problem → legal review.
- Products without a strain profile are not shown.

## Tests
`cd apps/api && DISPO_BASIC_USER=… DISPO_BASIC_PASS=… venv/bin/python scripts/menu_check.py` — 32 checks (incl. shelf-is-last, suggestions, time filter): whitelist,
Active-only, prices/sizes, 21+ gate, mood & type filters, deep links, phone/tablet/TV layouts, kiosk showcase, no JS/CSP errors.

## iPad / iPhone (2026-09-27)
Every iPad/iPhone browser — **Chrome included** — uses Apple's WebKit engine, so "it works in Chrome" on a computer says
nothing about an iPad. Photos looked wrong on an iPad: the images used `height: 100%` inside an `aspect-ratio` box,
which WebKit doesn't resolve there (draws the image full size). Fix: images are pinned to the box with
`position: absolute; top: 0; left: 0; width/height: 100%`, plus a padding-based fallback for iOS < 15 (no
`aspect-ratio`). Playwright's WebKit needs ~218 extra system packages (GStreamer, virtual display...) — deliberately
NOT installed on this production server; run WebKit tests from the laptop instead.
