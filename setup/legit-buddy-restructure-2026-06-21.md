# Legit Buddy — Project Restructure & Security Fixes

**Date:** 2026-06-21

## What happened
Full cleanup of the legit-buddy-private codebase to set a clean foundation before building new features. The project was treated as a fresh start — dead code removed, structure organized properly, security gaps closed.

## Why
The original code had everything mixed together at the project root, the entire frontend (1,919 lines of CSS/JS/HTML) embedded in a Python file, and two unauthenticated endpoints that could be abused. Before adding features or doing a UI redesign (this is now a paid product being pitched to a second dispensary), the foundation needed to be right.

## What changed

### New structure
```
app/           → FastAPI app (main.py + routers/)
pipeline/      → data pipeline scripts (scraper, enrich, parse_schedule)
scripts/       → dev utilities (build_preview, test_api, validate_schedule)
docs/          → data files + current static frontend (unchanged for now)
```

### Security fixes
1. `/api/menu/refresh` now requires an `X-Internal-Token` header. Returns 403 without it. Token stored in `.env` as `INTERNAL_TOKEN`.
2. CORS changed from `allow_origins=["*"]` to `["http://167.99.235.92"]`. Will update to domain once one is assigned.
3. Staff schedule photos (`schedule-june.jpg`, `schedule-may.jpg`) removed from git — they contain staff working hours and shouldn't be in version control.
4. Build output files (`dev.html`, `debug_api.json`, `Strain_Profiles_Updated.docx`) removed from git and gitignored.

### Dead code removed
- `generate_strain_doc.py` — one-time script with a hardcoded path to `/root/.claude/uploads/...`, already used and done.

### New additions
- `.env.example` — shows required environment variables without values
- `GET /api/strains` endpoint — returns `strains_enriched.json` for the frontend to use once decoupled
- `venv/` added to `.gitignore`

## Systemd service update
Changed from `uvicorn main:app` to `uvicorn app.main:app` with `PYTHONPATH=/opt/legit-buddy-private` so the app can find its own submodules.

```ini
Environment=PYTHONPATH=/opt/legit-buddy-private
ExecStart=/opt/legit-buddy-private/venv/bin/uvicorn app.main:app --host 127.0.0.1 --port 8002
```

## What I learned
- The original frontend (`docs/index.html`) was not fetching from the API at runtime — all product/strain data was baked in at build time by `build_preview.py`. The API endpoints existed but nothing called them. The upgrade (Phase 3) will flip this: live API fetches, no build step.
- FastAPI routers let you split endpoints across files cleanly. Each router file registers routes with `router = APIRouter()` and the main app includes them with `app.include_router(...)`.
- The `X-Internal-Token` header pattern is a simple way to protect admin-only endpoints without a full auth system. Token is a 32-byte hex string generated with `python3 -c "import secrets; print(secrets.token_hex(32))"`.

## Follow-up
- [ ] Install Playwright chromium: `venv/bin/playwright install chromium --with-deps`
- [ ] Set up cron job to replace GitHub Actions daily scrape
- [ ] Add real `ANTHROPIC_API_KEY` to `.env` when available
- [ ] Phase 3: decouple frontend from build step (extract CSS/JS from build_preview.py into static files, make them fetch from API)
- [ ] UI redesign — professional look, full dark/light mode
- [ ] Domain + SSL cert (certbot)
- [ ] Schedule image upload endpoint (PIN-protected)
