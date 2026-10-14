# dispo_menu: New & Sold out tile (2026-10-14, dev only, main 2ae6d3a)

## What it does
A staff tile (admins + employees, read only): what arrived on the live menu, what sold out, and what came back in
stock, over the last 1-10 days (store time; "1" = today). Category chips, each product's sizes/prices/THC with links
to the live-menu page, whether it has a profile; admins get a Generate button for unprofiled flower/vapes/concentrates.

## How it works
- Every menu refresh (5-min timer, store hours, and the Refresh button) already pulls the whole live menu. After it,
  `app/sync/menu_changes.py` `record_menu_changes` compares with table `menu_products` (every product ever seen,
  linked or not) and writes `menu_product_events` (kind new | sold_out | back, `at`). No extra store calls.
- Sweed's public menu drops a product when it sells out, so "left the menu" = sold out (or removed).
- Rules that keep it trustworthy (tested by `scripts/menu_changes_check.py`, free, rolled back):
  - first run only takes stock (2026-10-14 01:35 UTC: 54 products, 0 events);
  - gone < 15 min (`SOLD_OUT_AFTER`) and back = a glitch, no event; sold-out time = when it disappeared;
  - a whole category missing = probably a failed read, its products are left alone;
  - a sold-out product that returns = "Back in stock", not "New".
- API `GET /api/menu-changes/?days=N` (`routes/menu_changes.py`); page `apps/admin/src/pages/MenuChangesPage.tsx`.

## Limits
- Only knows changes since tracking started (the page says so).
- Product level, not size level (a 3.5g selling out while the 7g stays isn't shown yet).
- Overnight restocks are dated at the first refresh of the morning (the timer only runs in store hours).
- Foundation for the "Product History" placeholder tile (price/stock history).
