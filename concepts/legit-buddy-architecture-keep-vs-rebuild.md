# Legit Buddy — Architecture Review: What to Keep vs. Rebuild for a New Store

**Date:** 2026-07-30

## What it is

A review of `legit-buddy-private`'s architecture, done ahead of building the same system for
a second dispensary (the owner is paying for a second build — see `ROADMAP.md`). The question
being answered: when starting the new store's site, what should be reused as-is, what should be
rebuilt from scratch, and what security shortcuts shouldn't be copied forward.

## Current Architecture

```
Sweed POS ──(scraper.py)──► products.json ──┐
                                              ├──► FastAPI (app/) ──► docs/ (static, live-fetched by app.js)
Claude API ──(enrich.py)───► strains_enriched.json ──┘
```

- **Server pattern:** FastAPI + systemd (`legit-buddy.service`) + nginx reverse proxy on port
  8002, venv-isolated. Same pattern as `money_buddy` and `capitans_terps` on this droplet.
- **Backend:** `app/main.py` + 4 routers (`products`, `claude`, `schedule`, `deploy`) — thin,
  readable, ~165 lines total.
- **Pipeline:** `pipeline/scraper.py` (655 lines, hits Sweed's undocumented internal API
  directly), `pipeline/enrich.py` (209 lines, Claude-based strain enrichment + mood scoring),
  `pipeline/parse_schedule.py`.
- **Frontend:** `docs/app.js` (869 lines vanilla JS) + `docs/style.css` (499 lines), no
  framework, fetches `/api/products` + `/api/strains` at runtime.
- **Data:** flat JSON files (`products.json`, `strains_enriched.json`, `schedule.json`) — **no
  SQL database anywhere**. FastAPI routes `json.load()` these files straight off disk on every
  request (see `app/routers/products.py`). Cron commits them to git 4x/day as a crude
  versioning/backup mechanism.
- **Auth:** none for staff; a single shared `X-Internal-Token` header gates three admin
  endpoints (`/api/claude`, `/api/menu/refresh`, `/api/deploy`).

## Why it matters / how it works

Legit Buddy is now a template being built twice — once already live, once pending for store #2.
Anything reused as-is gets its flaws duplicated; anything rebuilt gets built once for both
stores if done before the second build starts. That makes this the right moment to separate
"worked, keep it" from "worked, but don't copy it forward."

### Worth keeping

| Piece | Why |
|---|---|
| Sweed scraper reverse-engineering (`pipeline/scraper.py`) | `storeid` in a header not the body, `last_store` cookie format, THC `max()` fix for the delta-9/total-THC array, SSR-vs-client-rendered category quirk — all undocumented, found via manual Network-tab inspection. Highest-value asset if store #2 also runs Sweed. |
| COA-based mood scoring engine (`enrich.py` + formula) | Position-weighted terpene scoring + Claude calibration rules (score ceilings, "no strain scores 8+ on 3+ moods") is a real product differentiator vs. marketing-copy-based competitors. |
| Write-once enrichment cache | Never re-paying Claude for strains already enriched. Good cost-control pattern, keep the idea and the JSON shape. |
| systemd + nginx + venv server pattern | Proven across 3 apps on this droplet already. |
| Cron replacing GitHub Actions | Exact schedule times instead of GitHub's 1-4hr runner delay (see `legit-buddy-page-limitations.md`). |
| `X-Internal-Token` pattern itself | Simple, appropriate for a low-stakes internal tool — keep the pattern even if scope is tightened per-endpoint. |

### Worth rebuilding, not carrying forward

| Piece | Issue |
|---|---|
| Flat JSON files with no DB, versioned via git | No SQL database anywhere — every request does a full `json.load()` of the whole file, and the cron job commits these files to git 4x/day forever as a substitute for real storage/versioning. 20+ data-only commits in 5 days already. A new build should use a real embedded DB (SQLite is the natural fit — zero extra infra, same droplet, but actual querying/indexing) instead of loading a whole JSON blob into memory on every request. |
| `docs/app.js` — 869-line single-file vanilla JS | Every roadmap feature (interactive strain profile UI, light/dark mode, product descriptions) means more imperative DOM code piled into one file. Building this twice means duplicating the cost — worth modularizing (or a lightweight framework) before the second build, not after. |
| `/api/deploy` self-deploy endpoint | Network-reachable endpoint running `git pull` + `systemctl restart`, gated by the *same* shared token as the AI proxy and refresh endpoint. One leaked token = deploy arbitrary committed code + spend the Anthropic key + trigger scrapes. Already contributed to one incident (the `strains_enriched.json` merge-conflict issue in `SESSION_NOTES.md`). For store #2, use distinct tokens per endpoint at minimum, ideally a signed webhook instead of a bearer-style header. |
| `/api/claude` generic proxy | Forwards the entire request body straight to Anthropic's Messages API using the server-side key, no constraint on model/params/cost. Anyone with the token can run arbitrary, arbitrarily expensive prompts against the bill. Fine internal-only; don't replicate as a generic passthrough. |
| Client-side "PIN protection" | Schedule PIN (`0420`) is checked in browser JS with a `sessionStorage` bypass — a UI speed bump, not access control, since the code and PIN both ship to the client. Don't describe it as "protected" to the owner, don't reuse for anything actually sensitive. |
| Public/private repo split (`legit-buddy-api` GitHub Pages + `legit-buddy-private` droplet) | Already caused real confusion — a whole doc (`legit-buddy-static-content-pattern.md`) exists just to explain that editing the public `index.html` directly is pointless because `build_preview.py` overwrites it. Unless there's a specific reason to want a free public mirror, store #2 should skip the "static build → later migrate to live API" phase and start on the droplet with live fetch from day one. |

### Already correctly discarded

`build_preview.py`'s single-build model, the old `generate_strain_doc.py` one-off script, and
the CORS-wildcard/no-auth state of the original endpoints were already cleaned up in the June 21
restructure (`legit-buddy-restructure-2026-06-21.md`) — nothing to redo there.

## What it does NOT do

- This doc does not record a decision that's been acted on yet — it's the input to one. No code
  changed as a result of this review.
- Does not cover the frontend UI/visual design itself (colors, layout) — only the code structure
  underneath it.

## Key commands

None — this is a review doc, not a setup doc. See `legit-buddy-restructure-2026-06-21.md` and
`legit-buddy-scraper-cron-2026-06-23.md` for the commands behind the pieces discussed above.
