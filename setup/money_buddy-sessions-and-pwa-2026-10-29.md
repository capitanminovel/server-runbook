# money_buddy: sessions, splits, deposits + mobile PWA (2026-10-29)

Commit `44364a8` in `capitanminovel/Transaction_Tracker`, deployed to `/opt/money_buddy`.

## What changed
- **Save session** (`POST /sessions`) replaces "Mark as paid". It:
  - freezes J/B/T totals into a new `sessions` tab
  - stamps every tagged expense, deposit and refund with `session_id` + `settled_at`
  - leaves untagged expenses open
- **Splits:** open a row and enter dollar amounts per person. They're stored as `tag=split`, `split=J:30.00;B:20.00`, and must add up to the amount (checked on the server).
- **Notes:** each transaction has a note (max 500 chars) that saves as you type.
- **Running total bar:** shows what J and B owe right now, recalculated in the browser on every tap.
- **Deposits:** card payments (`Type=Payment`) are now imported. Deposits and refunds show grayed out, can't be tagged, and are never counted (the user chose not to subtract refunds).
- **Pages:** Tag / Sessions / Totals, mobile-first, light and dark mode.
- **PWA:** installable on the phone. See [[concepts/pwa-and-service-workers.md]].
- **Security fixes:**
  - API calls without a login now get `401` JSON instead of an HTML redirect.
  - All bank and user text is HTML-escaped before rendering (`esc()` in `static/common.js`), since notes are typed by people.
  - Logout redirect fixed.

## Sheet layout
- **Sheet1 columns:** `... settled_at, type, note, split, session_id` (A–O). Filter on `session_id` to see a session.
- **`sessions` tab:** `session_id, saved_at, saved_by, tx_count, J, B, T, J_owes, B_owes`.
- **Backups:** `Sheet1_backup_2026-10-29` (before migration) and `Sheet1_backup_2026-08-27`.

## Migration (one-time, already applied)
```bash
cd /opt/money_buddy
venv/bin/python migrations/2026_10_29_sessions.py          # dry run
venv/bin/python migrations/2026_10_29_sessions.py --apply  # backup + write
```
- It turned the 3 past settle batches into sessions 1–3 and added back 11 skipped payments.
- Payments were assigned to sessions **by date**. The batch date ranges overlap by a day, so that assignment is approximate.
- **Order matters:** run the migration *before* restarting the new code. The new code creates an empty `sessions` tab on first use, and the migration treats an existing tab as "already ran" and does nothing.

## How it was tested without touching live data
- A test copy of the app ran on `:8010` with a fake in-memory `SheetsClient` subclass, seeded from a read-only snapshot of the sheet.
- API tests used an `itsdangerous`-signed test cookie.
- The UI was driven with Playwright (from the dispo-menu venv) at a 412×891 phone viewport.

## Deploy
```bash
cd /opt/money_buddy && git pull && systemctl restart money-buddy
```

## Follow-up
- `auth.py` uses `httpx.AsyncClient(verify=False)` for the Google token exchange, which turns off TLS certificate checks. Remove it and test login.
- Parked idea: a manual statement-total field to reconcile against the calculated total. Define the math first (payments, refunds, prior balance).
- Two people saving a session at the same moment could both get the same session number. That's acceptable with 3 users.
