# Money Buddy — `raw_transactions` archive tab

## What it is
A second tab on the Money Buddy Google Sheet ("Money Buddy's List",
ID `1PzV660AnPRlo1_qJhouQ6qAOS5g6s0J75kpu2S5J-Kg`) named **`raw_transactions`**.
It is an untouched log of every row from every file uploaded through the site,
kept purely for reference / reconciliation.

## Why it exists
The working tab (`Sheet1`) gets mutated by the app: rows are tagged, settled,
deduped on import, and `Type == "Payment"` rows are dropped entirely. When
per-person totals look wrong (e.g. the ~$1,000 discrepancy chased on 2026-08-27)
there was no pristine copy of the original data to check against. `raw_transactions`
is that copy.

## How it works
- On `POST /upload`, `main.py` copies **every parsed row verbatim** into
  `raw_transactions` **before** dedup / Payment filtering / tagging.
- Each archived row gets 3 extra columns appended: `transaction_id` (same
  `md5(date|merchant|amount)` hash Sheet1 uses, so rows line up across tabs),
  `imported_at` (UTC ISO timestamp), `source_file` (uploaded filename).
- Header order is **fixed on first upload** to the standard Chase export columns:
  `Transaction Date, Post Date, Description, Category, Type, Amount, Memo` + the 3
  above. Later uploads align to that header; unmapped source columns are dropped.
- Archiving is wrapped in try/except — if it fails, the normal import still runs
  and the `/upload` response reports `archived: 0`.

## What it does NOT do
- The app never **reads** `raw_transactions` — every Sheets call in `sheets.py`
  hardcodes `Sheet1!`. Totals, the transaction list, tagging, and settling are
  100% unchanged.
- It does not dedup — the same transaction uploaded 3 times appears 3 times here
  (distinguish by `imported_at` / `source_file`).
- It is not a backup of Sheet1's tags/settlement state. That's a separate
  one-off tab: `Sheet1_backup_2026-08-27`.

## Code
- `sheets.py`: `RAW_TAB` constant, `_tab_titles()`, `ensure_raw_tab()`,
  `append_raw_rows()`.
- `main.py`: archive block in `upload_transactions()`, `archived` count in the
  response, `from datetime import datetime, timezone`.
- `static/app.js`: upload status text shows `N archived to raw`.
- Commit `fd64f9f` on `main` in `github.com/capitanminovel/Transaction_Tracker`.

## Key commands
```bash
systemctl restart money-buddy.service      # required after code changes (no --reload)
journalctl -u money-buddy.service -f        # watch uploads land
# inspect the tab without the app:
cd /opt/money_buddy && ./venv/bin/python -c "
from dotenv import load_dotenv; load_dotenv('/opt/money_buddy/.env')
from sheets import SheetsClient, RAW_TAB
c=SheetsClient()
print(len(c.sheet.values().get(spreadsheetId=c.sheet_id, range=f\"'{RAW_TAB}'!A:A\").execute().get('values',[])), 'rows')"
```

## Backfill (done 2026-08-27)
- Full history uploaded as `Chase8827_Activity_20260827 (1).csv` (477 data rows,
  05/12 → 08/25/2026). Response: `imported 11, 11 payments skipped, 455 dupes`.
- `raw_transactions` now holds all 477 rows — complete reference.
- The 11 "imported" rows were genuine pre–Money Buddy transactions (05/12–05/18,
  before the app's first import). Per user request they were **deleted from
  Sheet1** (script: scratchpad `remove_new.py` — deletes rows whose
  transaction_id is absent from `Sheet1_backup_2026-08-27`, aborts if any target
  is tagged/settled). Sheet1 back to 452 rows, identical to the backup.
- 455/466 non-payment rows deduped cleanly → the export's date/amount formatting
  matches stored data; dedup is reliable for this source.

## Follow-up
- 69 untagged Aug 9–25 transactions still sit in Sheet1 awaiting T/B/J tags.
- `Sheet1_backup_2026-08-27` retained as a safety net; safe to delete later.
- Still open: the ~$1,000 / $4,584.71 reconciliation (candidate: `CL *Chase Travel`
  -$1,118.40, tagged B, settled 2026-08-12 01:49).
- App code committed locally (`fd64f9f`) but **not pushed** — `git push` from
  `/opt/money_buddy` when ready.
