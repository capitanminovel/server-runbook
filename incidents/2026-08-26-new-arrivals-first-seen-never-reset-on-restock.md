# New Arrivals / Sold Out Silently Broke After Long Stock Gaps (South Metro)

**Date:** 2026-08-26
**Repo:** `mnlegitdev/legit-cannabis-south-metro-menu`

## What happened

User reported New Arrivals wasn't registering products, and specifically flagged that "Runtz 7g" was a real new arrival not showing, and that Lit OG had genuinely been sold out without the site ever reflecting it. Both turned out to be correct — confirmed by walking `docs/products.json` through every daily-scrape commit since 08-20 rather than trusting a single snapshot:

- **Runtz** (flower) sat out of stock for 58 days (last in stock 2026-06-29, tracked under a legacy text-hash key with no Sweed id, weight `4g`). It restocked at 21:37 UTC on 2026-08-26 with a different tier structure (`eighth`/`quarter`, the "7g" the user saw) and a Sweed id it hadn't had before.
- **Lit OG** (flower + pre-roll) sat out of stock for 18 days (last in stock 08-08 / 07-19) and restocked in that same 21:37 UTC run.
- Neither showed up as "New" once restocked, and Lit OG's 18-day absence never appeared under "Sold Out" beyond its first 2 days — it just vanished from the site with no signal either way.

## Why

`merge()` in `scraper.py` computed `first_seen` as "the first time this key was ever recorded," full stop — it only ever got reset on a category change, never on a stock transition. So:

- A product that goes out of stock and comes back keeps its original `first_seen` forever, no matter how long the gap. "New Arrivals" (`build_preview.py`, `NEW_DAYS = 3`, filters on `first_seen`) can never see a restock as new.
- The legacy-key migration path (bridges a product from a text-hash key to a Sweed-id key — see `[[2026-07-15-preroll-false-new-badge-product-key-rewrite]]`) made this worse for id-less products: it silently carries the ancient `first_seen` across the migration with no gap check at all.
- Separately, "Sold Out" (`SOLD_DAYS = 2`, filters on `last_seen`) only shows items that went out of stock in the last 2 days. A product out of stock longer than that isn't wrong, exactly, but there's no "still gone, been gone a while" state — it just disappears from the entire site.

Root design gap: `first_seen` meant "ever recorded," but "New Arrivals" needs "recently became available" — a strain gone for weeks and restocked is new to a customer/budtender even if the system technically saw it months ago.

## Fix

In `scraper.py`, added a restock check applied on both the normal same-key path and the legacy-key migration path:

```python
RESTOCK_GAP_DAYS = 2  # matches build_preview.py's SOLD_DAYS

def _is_restock(old, new, now):
    if old.get("in_stock", True) or not new.get("in_stock", False):
        return False
    return _days_since(old.get("last_seen", ""), now) > RESTOCK_GAP_DAYS
```
If a product was out of stock and comes back after more than `RESTOCK_GAP_DAYS`, `first_seen` resets to now instead of carrying over the old value. A same-day stock-check hiccup (out then back in within one run) does not reset it, so New Arrivals won't spam on ordinary noise.

Verified against real data before deploying: Runtz (58d gap) and Lit OG (18d gap) both correctly reset; a synthetic 1-day gap and a continuously-in-stock product both correctly did *not* reset.

Also manually backdated the two already-affected live records in `docs/products.json` (`first_seen` → now for Runtz, LIT OG, PR Lit OG) and re-ran `build_preview.py` so the live site reflected it immediately instead of waiting for the next scrape's drift to self-correct — same one-time-backdate pattern as the prior incident.

Commit: `9c786d5`.

## What I learned

- "First time ever seen" and "recently became available" are different concepts and this codebase only tracked the former — worth checking any other place age-based state (`first_seen`/`last_seen`) drives a UI filter for the same kind of gap.
- Don't trust a single git snapshot when auditing time-based logic — walking the actual commit history was what surfaced the real restock timeline; a snapshot alone said "nothing's new," which was wrong.

## Follow-up

- **Not yet ported to `dinky-dope-menu`.** Per `[[2026-07-15-preroll-false-new-badge-product-key-rewrite]]`, that repo is "near-identical code" to this one and has independently needed the same fixes twice before (product-key identity, orphaned-entry sold-out false positives). This restock/`first_seen` gap almost certainly exists there too and hasn't been checked yet.
- Same standing note as the prior incident: two independently-maintained near-identical repos means every `scraper.py` bug gets found and fixed twice. Still not acted on.
