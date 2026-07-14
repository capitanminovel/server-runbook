# mnlegitdev: The Real Owner of the Dispensary Menu Repos

**Date:** 2026-07-14

## What this is

There are two separate sets of GitHub repos for the same two dispensary menu tools, and only one set is actually live:

| Purpose | Live repo (mnlegitdev) | Stale pre-handoff copy (capitanminovel) |
|---|---|---|
| MN Legit Cannabis South Metro menu | `mnlegitdev/legit-cannabis-south-metro-menu` | `capitanminovel/legit-buddy-api` |
| Dinky Dope (Dinkytown) menu | `mnlegitdev/dinky-dope-menu` | `capitanminovel/dinky-buddy-api` |

`capitanminovel` has **write** collaborator access to both `mnlegitdev` repos (confirmed via `gh api repos/mnlegitdev/<repo>/collaborators/capitanminovel/permission`).

## Why it matters

The `capitanminovel` copies are **not** git forks of the `mnlegitdev` ones — they're fully independent repos (`gh api repos/capitanminovel/<repo> -q '.fork, .parent.full_name'` returns `false` / empty for both) that happen to contain near-identical code, left over from before ownership was handed off to `mnlegitdev`. Critically, they are **not inert** — both still have their own live GitHub Pages sites and their own scheduled Actions workflows (`daily-scrape.yml`) running several times a day, scraping Sweed and committing updates independently of the mnlegitdev repos.

This means: editing the wrong set silently does nothing useful (or worse, creates two menu pages that drift out of sync with each other, one of which nobody is actually looking at). This actually happened in this session — the "Terpenes & Cannabinoids 101" staff guide feature was built and pushed to the `capitanminovel` copies first, before the user clarified `mnlegitdev` was the real owner, requiring the same work to be ported over via `git diff` + `patch --fuzz`.

## What to do going forward

- Default to the **mnlegitdev** repos for any dispensary menu / staff guide work.
- If a request references "legit-buddy-api" or "dinky-buddy-api" by the old name, confirm whether they mean the live `mnlegitdev` repo before assuming.
- As of this writing, the `capitanminovel` copies have **not** been archived or had their workflows disabled — that's a decision still pending with the user. Don't assume they're safe to ignore entirely without checking first.
- Both sets of repos share the exact same `build_preview.py`-driven static content pattern — see `concepts/legit-buddy-static-content-pattern.md`. That mechanism applies identically regardless of which owner's copy you're in.

## Key commands

```bash
# Check who really owns/controls a repo and whether it's a fork
gh api repos/<owner>/<repo> -q '.fork, .parent.full_name'

# Check your own permission level on someone else's repo
gh api repos/<owner>/<repo>/collaborators/<your-username>/permission

# Find all repos you're a collaborator on under a specific org/user
gh api /user/repos --paginate -q '.[] | select(.owner.login=="<org>") | .full_name'
```
