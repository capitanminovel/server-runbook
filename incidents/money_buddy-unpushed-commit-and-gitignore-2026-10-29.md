# money_buddy: unpushed commit + venv not ignored (2026-10-29)

## What happened
During a routine health check, `/opt/money_buddy` was found to be 1 commit ahead of `origin/main`:
`fd64f9f` "Archive every uploaded row to a reference-only raw_transactions tab" (from 2026-08-27).
It was running live but had never been pushed, so the server held the only copy for ~2 months.
`git status` also showed `venv/` as untracked.

## Why
- The commit was made on the server and the push step was skipped.
- `.gitignore` listed `.venv/` (with a dot), but the real virtualenv folder is `venv/` (no dot), so git didn't ignore it.

## Fix
```bash
cd /opt/money_buddy
echo "venv/" >> .gitignore
git add .gitignore
git commit -m "Ignore venv/ (actual virtualenv folder name)"
git push
```
`main` now matches `origin/main` (`6d0d119`).

## What I learned
- "Ahead N" in `git status -sb` means local commits GitHub doesn't have. If the server dies, those commits are lost.
- A `.gitignore` entry has to match the real folder name exactly. `.venv/` does not cover `venv/`.
- If the venv isn't ignored, a habitual `git add .` would commit thousands of server-specific package files.

## Follow-up
- Checked the same day: `capitans_terps` is clean and in sync. `/opt/dispo-menu*` are deploy targets, not git repos (their code comes from the source repo via `deploy.sh`), so nothing to check there.
