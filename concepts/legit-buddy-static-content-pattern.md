# legit-buddy-api: How Static Modal Content (Staff Guide, Terpenes 101) Is Generated

**Date:** 2026-07-14

## What it is

`legit-buddy-api` (the public GitHub Pages repo, not the droplet-hosted `legit-buddy-private`) publishes a single static file: `docs/index.html`. That file is fully regenerated from scratch by `build_preview.py` on every scheduled scrape (5x/day via `.github/workflows/daily-scrape.yml`), which then commits and pushes the rebuilt file back to `main`.

Most of that page is genuinely data-driven — product cards, mood scores, terpene lists — pulled from `docs/products.json` (Sweed scrape) and `docs/strains_enriched.json` (Claude-generated strain profiles). But some sections, like the **Staff Guide modal** and the new **Terpenes & Cannabinoids 101 modal**, are pure hand-authored reference text with no connection to the scrape data at all.

## Why it matters / how it works

`build_preview.py` is a Python script that renders the entire page as one giant f-string. Any HTML/CSS/JS that needs to appear in the final page — including static, never-changing content like the Staff Guide's explanation of how the tool works — has to be written directly inside that f-string, as a JS template literal assigned to `.innerHTML` at click-time (see `openStaffGuide()` around line 993).

Because it's inside a Python f-string, every literal `{` or `}` in the embedded CSS/JS has to be escaped as `{{` / `}}` or Python's `.format()`-style parsing breaks.

**This means:** editing `docs/index.html` directly to add or change static content is pointless — the next scheduled Action run overwrites it with whatever's in `build_preview.py`. Any content change (including plain English copy edits to the Staff Guide) must be made in `build_preview.py` and then either committed directly (content shows up on the next scheduled build) or the script can be run locally (`python3 build_preview.py`) to regenerate `docs/index.html` immediately and commit both files together.

This was already called out in the repo's own `HANDOFF.md` ("Manual edits to `docs/index.html` don't survive") but is easy to miss since the file *looks* like a normal static site you could hand-edit.

## How it's set up on this server

Added 2026-07-14: a second static modal, "🧪 Terpenes & Cannabinoids 101", next to the existing "📖 Staff Guide" button in `build_preview.py`. It follows the exact same pattern as the Staff Guide modal — same `.sg-guide-*` CSS classes, same overlay/box structure, a new `openTerpeneGuide()` / `closeTerpeneGuide()` JS function pair, and the new modal's close handler added to the existing `Escape` key listener.

Content: quick-reference cards for the 8 terpenes already used by the mood filter (Myrcene, Limonene, Caryophyllene, Linalool, Pinene, Terpinolene, Humulene, Ocimene) plus THC/CBD/CBG/CBN/THCV and a short plain-language note on how cannabinoids modulate THC's effect. Written for staff to read off to customers quickly — not the deep research version (that's `docs/terpenes_research.md`, which stays as the citation-heavy backing reference).

Source content pulled from a private cannabis knowledge-base repo (`capitanminovel/Canna_Buddy`, `cannabis-kb/06-chemistry/`) which is the only fleshed-out section of that KB so far — everything else in it (cultivation, plant health, genetics, etc.) is still just outline stubs.

## What it does NOT do

- Does not pull from `products.json` / `strains_enriched.json` — this content is 100% static and will never update on its own.
- Does not persist if edited in `docs/index.html` directly — always edit `build_preview.py`.
- Does not require an Anthropic API call to render — unlike strain profile enrichment, this is plain hardcoded text, free to regenerate as often as needed.

## Key commands

```bash
# Regenerate docs/index.html locally after editing build_preview.py
cd /root/legit-buddy-api
python3 build_preview.py

# Verify script has no syntax errors before running
python3 -c "import ast; ast.parse(open('build_preview.py').read())"
```
