# dinky-buddy-api — UI Patterns & Build Notes

**Date:** 2026-06-24  
**Repo:** `C:\Users\josem\dinky-buddy-api` / `github.com/capitanminovel/dinky-buddy-api`

---

## Context

`dinky-buddy-api` is the same stack as `legit-buddy-api` (static build — no live API). The menu is built by running `build_preview.py`, which reads `docs/products.json` and `docs/strains_enriched.json` and writes `docs/index.html`. GitHub Pages hosts it. A daily GitHub Actions workflow scrapes and rebuilds automatically.

**Important:** any change to the visible HTML must be made in `build_preview.py` (the template), not directly in `docs/index.html`. If you edit `index.html` directly and the daily build runs, your changes are overwritten.

---

## What Was Done

### 1. Hiding UI elements (comment-out pattern)

We hid three things without deleting them — wrapping in `<!-- -->` so they're easy to restore:

- **Export buttons** (Available Now / Master Cache) — hidden but Staff Guide button kept
- **Legend bar** (Indica/Sativa/Hybrid color key below the export bar)
- **Schedule tab** in the tab bar

**Where to make these changes:** in `build_preview.py`, not `index.html`.

The export bar and legend are in the `html = f"""..."""` string around lines 588–614. The schedule tab is added dynamically on line ~207:
```python
tab_btns += '\n<!-- <button class="tab sched-tab" ...>📅 Schedule</button> -->'
```

To restore any of these, remove the `<!-- -->` wrappers in `build_preview.py` and rebuild.

---

### 2. Moving strain type badge from image overlay to above product name

**Before:** Indica/Sativa/Hybrid badge was absolutely positioned inside `.card-img` (overlaid on the photo).

**After:** badge sits in `.card-body`, between the brand line and the product name.

**Change in `build_card()` in `build_preview.py`:**

```python
# Old: badges div included both age badge and strain badge
badges = f'<div class="badges">{age_b}{strain_b}</div>' if (strain_b or age_b) else ""

# New: badges div only contains the age badge (New Today / 2d ago)
badges = f'<div class="badges">{age_b}</div>' if age_b else ""
```

And in the card template:
```python
# Old:
<div class="card-img">{img}{badges}{potency}...</div>
<div class="card-body">
  {brand_h}
  <div class="card-name">...

# New:
<div class="card-img">{img}{badges}{potency}...</div>
<div class="card-body">
  {brand_h}
  {strain_b}           ← moved here
  <div class="card-name">...
```

To apply this to legit-buddy-api, the same pattern applies — find `build_card()` in its `build_preview.py` (or `scripts/build_preview.py`) and make the same split.

---

### 3. Strain type filter chips (Indica / Sativa / Hybrid / CBD)

Added a row of filter chips in the mood bar that filter cards by strain type, working alongside category tabs and mood filter.

**Three parts:**

**a) `data-strain` attribute on each card** (in `build_card()`):
```python
strain_key = (p.get("strain_type") or "").lower().split("(")[0].strip()
# "Hybrid (Indica)" → "hybrid", "Sativa" → "sativa"

return f'<div class="card" data-strain="{strain_key}" ...>'
```

**b) HTML in mood bar** (inside the `html = f"""..."""` string, between the mood chips row and the search row):
```html
<div class="type-filter-row">
  <span class="type-filter-label">Type</span>
  <div class="type-chips" id="typeChips">
    <button class="type-chip on" data-type="" onclick="filterType(this)">All</button>
    <button class="type-chip" data-type="indica" onclick="filterType(this)">Indica</button>
    <button class="type-chip" data-type="sativa" onclick="filterType(this)">Sativa</button>
    <button class="type-chip" data-type="hybrid" onclick="filterType(this)">Hybrid</button>
    <button class="type-chip" data-type="cbd" onclick="filterType(this)">CBD</button>
  </div>
</div>
```

**c) JS** — three additions:
```javascript
// State variable (alongside activeMood, activeCat, activeSearch)
let activeType = '';

// In applyFilters(), add typeOk check
const typeOk = !activeType || (card.dataset.strain || '') === activeType;
const visible = catOk && typeOk && moodOk && searchOk;  // add typeOk here

// New function (after clearMood)
function filterType(btn) {
  document.querySelectorAll('.type-chip').forEach(c => c.classList.remove('on'));
  btn.classList.add('on');
  activeType = btn.dataset.type;
  applyFilters();
}
```

CSS uses the existing strain color variables (`--indica`, `--sativa`, `--hybrid`, `--cbd`) for the active chip state.

---

### 4. Fixing edibles in the modal

**Problem:** `enrich_strains.py` was skipping edibles and writing a stub entry with blank `therapeutic`, `aroma`, `misc` fields. So tapping an edible card showed almost nothing in the modal.

**Fix — two parts:**

**a) Copy enrichment from legit-buddy-api for matching products.**  
Product keys are deterministic (MD5 of `"name-brand"`), so the same product at two stores has the same key. Run:

```python
import json

with open('docs/strains_enriched.json', encoding='utf-8') as f:
    target = json.load(f)
with open('../legit-buddy-api/docs/strains_enriched.json', encoding='utf-8') as f:
    source = json.load(f)

# Find stubs (edibles that were skipped)
stubs = [k for k, v in target.items() if 'distillate edible' in v.get('lineage', '')]

copied = 0
for key in stubs:
    if key in source:
        target[key] = source[key]
        copied += 1

with open('docs/strains_enriched.json', 'w', encoding='utf-8') as f:
    json.dump(target, f, indent=2, ensure_ascii=False)

print(f"Copied {copied} entries")
```

**b) Fix `enrich_strains.py` to enrich edibles going forward** (not stub them):
```python
# Old (lines ~181-194):
if cat in EDIBLE_CATEGORIES:
    print(f"  → {product.get('name')} — edible, skipping enrichment")
    existing[key] = {"lineage": "N/A — distillate edible", ...blank fields...}
    continue

# New:
if cat in EDIBLE_CATEGORIES:
    print(f"  → {product.get('name')} — edible, enriching with dosing focus")
    # falls through to the normal enrich_product() call below
```

The `PROFILE_PROMPT` already has an instruction: *"For edibles adapt accordingly — no genetic lineage needed, focus on dosing guidance."* So Claude handles them correctly when actually called.

**c) Hide the "N/A — distillate edible" lineage** in the modal so old stubs don't show that string:
```javascript
// In buildSgCard rows array, change lineage line to:
s.lineage && s.lineage !== 'N/A — distillate edible'
  ? `<div class="sg-row"><strong>Lineage:</strong> ${s.lineage}</div>`
  : '',
```

**d) Show `p.description` as a fallback** when `misc` is empty (catches any remaining unenriched products):
```javascript
(!s.misc && p.description)
  ? `<div class="sg-row"><strong>About:</strong> ${p.description}</div>`
  : '',
```

---

## Earlier Session Fixes (same day, before the UI work above)

These were done in a prior session and are captured here for completeness.

### 5. Scraper — storeid header + last_store cookie (WAF bypass)

**Problem:** The Sweed POS API at `/_api/Products/GetProductList` was returning empty or blocked responses. The direct HTTP strategy was failing because the request was missing required headers that the browser sends automatically.

**Fix:** Added `storeid` as a request header and a `last_store` cookie containing a URL-encoded JSON blob with the store identity. This matches what the Sweed frontend sends and gets past the WAF check.

```python
STORE_ID = 609  # Dinkytown store ID (from browser Network tab)

import urllib.parse as _urlparse, json as _json
_last_store = _urlparse.quote(_json.dumps({
    "id": STORE_ID,
    "url": f"https://{STORE_DOMAIN}/{STORE_SLUG}",
    "routeName": f"/{STORE_SLUG}",
}))

HEADERS = {
    ...
    "storeid": str(STORE_ID),
    "ssr": "false",
    "Cookie": f"last_store={_last_store}; swa_Common/isAgeChecked=true",
}
```

**How to find the store ID for a new store:** open the store's menu in Chrome, open DevTools → Network tab, filter for `GetProductList`, click the request, look at the request headers for `storeid` or the request body for `storeId`.

**Note:** Even with this, the WAF still blocks the direct API call in GitHub Actions (different IP, no browser session). Playwright is the real fallback that runs in production — the direct API works locally but usually fails in CI.

---

### 6. UTF-8 BOM fix

**Problem:** Files copied from another project had a UTF-8 BOM (`﻿`, `﻿`) at the start. Python's `json.load()` choked on it, and the em-dash characters in comments appeared as garbled multi-byte sequences (`â€"` instead of `—`).

**Fix — two parts:**

**a) Strip BOM from JSON files** (`products.json`, `strains_enriched.json`, `schedule.json`, `debug_api.json`) — open in a text editor set to UTF-8 without BOM and re-save, or run:
```python
for path in ['docs/products.json', 'docs/strains_enriched.json', ...]:
    text = open(path, encoding='utf-8-sig').read()  # utf-8-sig strips BOM on read
    open(path, 'w', encoding='utf-8').write(text)
```

**b) Make `load_db()` in `scraper.py` BOM-tolerant** as a permanent safeguard:
```python
def load_db() -> dict:
    if DATA_FILE.exists():
        with open(DATA_FILE, encoding='utf-8-sig') as f:  # utf-8-sig = strip BOM if present
            return json.load(f)
    return {...}
```

`utf-8-sig` is safe to use even when there's no BOM — it just reads normally in that case.

---

### 7. Rebrand — yellow/black color theme + Dinky Dope logo

This project was initially copied from legit-buddy-api and needed full rebranding for Dinky Dope.

**Colors changed:** brand green (`#1a7a4a`) replaced with Dinky Dope gold (`#C9A000` light mode, `#F5C228` dark mode). Updated in CSS variables, mood chip active states, schedule today highlight, terpene pill colors, and the top announcement bar.

**Logo:** GIF mascots replaced with the official Dinky Dope logo image pulled from `dinkydope.com`.

**Text/filenames:** all "legit", "Legit", "south-metro", "legit-buddy" references replaced with "dinky", "Dinky Dope", "dinkytown", "dinky-buddy". This includes the `.docx` export filenames (`dinky-available-guide.docx`, `dinky-master-guide.docx`).

**Where the brand color is set** in `build_preview.py`:
```css
:root {
  --brand: #C9A000;   /* light mode */
}
body.dark {
  --brand: #F5C228;   /* dark mode */
}
```

To re-theme for another store, change these two hex values and the logo `src`.

---

## Applying These Changes to legit-buddy-api

legit-buddy-api has the same `build_preview.py` structure. The JS patterns (type filter, modal rows) are identical. The main difference is file paths:

| Thing | dinky-buddy-api | legit-buddy-api |
|---|---|---|
| Build script | `build_preview.py` | `scripts/build_preview.py` |
| Enrichment script | `enrich_strains.py` | `pipeline/enrich.py` |
| Products data | `docs/products.json` | `docs/products.json` |
| Strains data | `docs/strains_enriched.json` | `docs/strains_enriched.json` |

After any change to `build_preview.py`, run `python build_preview.py` to regenerate `docs/index.html`, then commit and push.
