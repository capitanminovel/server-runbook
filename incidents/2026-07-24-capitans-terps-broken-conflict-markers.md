# Capitan's Terps site broken — live git conflict markers

## What happened
`grow.withcapitan.com` (Capitan's Terps, `/opt/capitans_terps/`) was serving raw
git conflict markers (`<<<<<<< Updated upstream`, `=======`, `>>>>>>> Stashed
changes`) as literal text inside the page HTML. The site returned 200 but was
visibly broken.

## Why
A previous Claude session had been mid-refactor, moving `static/index.html`
off one giant inline `<style>`/`<script>` block onto external
`static/css/app.css` + `static/js/*.js` files (which already existed and were
more feature-complete — they included a lightbox and strain-lineage display
the old inline script never had). That refactor hit a `git stash pop`
conflict. The session got partway through resolving it by hand — even left a
stray custom `DELETE_MARKER_START` sentinel mid-edit — and was then
interrupted before finishing. The unresolved conflict markers were left
sitting in the file and served as-is (`git status` showed `main.py` and
`static/index.html` as "both modified" / unmerged).

Separately, `main.py`'s conflict had already been resolved correctly on disk
(same refactor: monolithic file → `db.py` + `routers/`) but never staged or
committed. And the core app modules the running app actually depends on —
`db.py`, `routers/`, `static/css/`, `static/js/` — had **never been tracked in
git at all**, so the repo alone couldn't have rebuilt the running app.

## Fix
- Compared the "Updated upstream" (HEAD) vs "Stashed changes" sides and
  confirmed the external CSS/JS files were the complete, correct target
  state (functions in the old inline `<script>` all existed in
  `static/js/*.js`, plus extras).
- Removed the obsolete inline `<style>`/`<script>` blocks and the conflict
  markers from `static/index.html`, keeping the `<link>`/`<script src>` tags
  to the external files.
- Restarted `capitans-terps.service`; verified `grow.withcapitan.com` returns
  clean HTML.
- Added `.gitignore` (`.env`, `venv/`, `__pycache__/`, `data/`) and committed
  `db.py`, `routers/`, `static/css/app.css`, `static/js/*.js` — files the app
  needs to run that were previously untracked.
- Pushed to `origin/main` (commit `7d4c1bf`).

## What I learned
- `git status` unmerged/"both modified" paths are worth checking after any
  interrupted session — a half-finished conflict resolution can leave broken
  markers live in production silently (no crash, just wrong output).
- Always diff `git ls-files` against what a service's entrypoint actually
  imports — untracked-but-required files are a silent single point of
  failure.

## Follow-up
None outstanding. Repo now matches what's running.
