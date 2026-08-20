# Ported "New Arrivals / Sold Out" tabs from legit-buddy-api to both mnlegitdev live repos

**Date:** 2026-08-20

## What happened

`capitanminovel/legit-buddy-api` had a merged commit (`f74e673`, Aug 12) that changed how the
"✨ New in the Last 3 Days" and "🚫 Sold Out" sections render: instead of always being dumped
above the product grid whenever "All Products" was selected, they became their own tabs
(`✨ New Arrivals`, `🚫 Sold Out`), shown only when selected. That commit had never been ported
to the two repos that actually run the live menus: `mnlegitdev/legit-cannabis-south-metro-menu`
and `mnlegitdev/dinky-dope-menu` (see [[mnlegitdev-repo-handoff]]).

Ported it to both via `git show f74e673 -- build_preview.py README.md STAFF_GUIDE.md > patch`
+ `git apply`. It applied cleanly to `build_preview.py` in both repos and to the docs in
`legit-cannabis-south-metro-menu`; `dinky-dope-menu`'s README/STAFF_GUIDE wording had drifted
enough that the doc portion of the patch had to be hand-edited instead.

## Why

The three repos (`legit-buddy-api` plus the two `mnlegitdev` live repos) share the same
`build_preview.py`-driven template pattern but are **not** forks of each other — feature work
done in one doesn't propagate anywhere else automatically. This is the same failure mode
already documented in [[mnlegitdev-repo-handoff]]: a fix lands in the wrong (or a
non-canonical) copy and the live sites don't get it unless someone notices and ports it by hand.

## Fix

For each of the two live repos:

```bash
cd /root/<repo>
git fetch origin
git apply --include=build_preview.py new-sold-tabs.patch   # + README/STAFF_GUIDE where wording matched
python3 build_preview.py                                    # rebuild docs/index.html from current data
git add build_preview.py README.md STAFF_GUIDE.md docs/index.html
git commit -m "Move New Arrivals and Sold Out into their own tabs"
git rebase origin/main       # local clone was stale; docs/index.html conflicted (generated file)
python3 build_preview.py     # regenerate docs/index.html from the now-current products.json instead of merging it
git add docs/index.html && git rebase --continue
git push
```

The only real conflict on rebase was `docs/index.html` — expected, since it's a generated
artifact and both sides had rebuilt it from different data snapshots. The right move is to
never hand-merge a generated file: take either side, then rerun the generator against
whatever `products.json` ends up on disk after the rebase, and stage that.

## What I learned

- Both `mnlegitdev` repos run their own independent scrape → commit → push cycle (their own
  GitHub Actions, same pattern as `legit-buddy-api`'s `daily-scrape.yml` per
  [[mnlegitdev-repo-handoff]]), so a local clone on this droplet goes stale within hours and
  must be fetched/rebased immediately before pushing, every time.
- `docs/index.html` is fully generated from `products.json` + `build_preview.py` — never worth
  diffing/merging by hand, just rebuild it after resolving everything else.
- After porting, whether the new tabs are visible depends on live data (products newer than
  `NEW_DAYS`/`SOLD_DAYS`), not on whether the code is correct — verified the port was correct
  by comparing against `legit-buddy-api`'s own already-live build, which did have qualifying
  products and did show the tabs.

## Follow-up

- None outstanding — both repos are pushed and will pick up the new tab behavior next time
  their own scrape produces a new-arrival or sold-out item.
