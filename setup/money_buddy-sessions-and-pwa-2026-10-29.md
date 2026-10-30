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

## Statement check (added later the same day, commit `6795985`)
- A card at the top of the Tag page. Type the statement's **charges** total and it's compared to the open list.
- **What it compares against:** charges − refunds ("net"). Gross charges also count as a match, because Chase's summary lists Purchases before credits are applied. Payments are shown but never counted.
- **Shared, not per-device:** stored in a new `settings` tab (`statement_total`), so everyone sees the same number.
- **When you save a session,** the value is copied to `sessions.statement_total` and cleared for the next statement.

## Login hardening (commit `e0210aa`)
- **Removed `verify=False`** from the Google token and user-info calls. With it, the server didn't check that it was really talking to Google, so the sign-in exchange could be intercepted. Verified TLS to Google works from the venv.
- ~~OAuth `state` cookie~~: **reverted** (commit `c50e807`) because it broke phone login. See `incidents/money_buddy-login-broken-by-oauth-state-2026-10-29.md`.
- **Dropped `access_type=offline`.** It asked Google for a refresh token the app never uses.
- **Session cookie is now `Secure`** (only ever sent over HTTPS).
- **Test:** sign out and back in on the phone once.

## Statement periods + fresh start (commit `f93658e`)
- **Post dates:** uploads now store Chase's **Post Date**. Statements are cut by post date, not purchase date.
- **Statement period:** a statement can have a start and end date (kept in the `settings` tab). The statement check, running total, To-tag list and Save session only count transactions posted inside it; the rest stay open, labeled "Earlier statement" or "Next statement".
- **Find:** type an amount (or tap "Use last payment"). The app tries every unbroken run of dates and looks for one where charges − refunds equal the amount: exact matches first, otherwise the closest 3. Trying date ranges instead of arbitrary combinations means there's normally one clear answer.
- **Sessions** now record `statement_total`, `period_start` and `period_end`.
- **Fresh start:** `migrations/2026_10_29_fresh_start.py --apply` copied Sheet1 (463 rows) and sessions (3 rows) to `*_archive_2026-10-29` tabs, then emptied the working tabs. `raw_transactions` was left untouched.
- **Limit:** Find only matches if the payment was the full statement balance. Partial payments, interest or fees mean no exact match.

## Balance tab: who paid, running balance, payment plans
- **Who paid:** card payments have a `paid_by` column (J / B / joint / none), set from a "Who paid?" picker on the Tag page. A joint payment credits half to each person.
- **Starting point:** `ledger_start` in the `settings` tab is the date the card was last at $0. `/balance` groups charges into stretches that end at each payment (by post date) and tracks paid − share per person. The settled result is the transfer between J and B; the stretch after the last payment is the open bill.
- **Payment plans:** stored as JSON under `plans` in the `settings` tab. Each line is one of:
  - a live figure (the settled transfer, or a person's share of the open bill)
  - a share of one transaction (e.g. half a flight)
  - a fixed amount

  Any line can also have an adjustment and says who pays it. The page shows each plan with a total per person.
- **Sharing:** "Share as text" opens the phone's share sheet; "Copy" copies the same text.
- **Cleanup:** removed unused code and the one-time migration scripts (they're in git history).
- Household amounts and names are kept out of this runbook on purpose, because this repo is public.

## Native app decision
- No APK. Chrome's "Install app" already installs a real Android package (a WebAPK) on the phone.
- A sideloaded APK wrapping the site (a TWA) would need about 2 GB of Android build tools on the server, a signing key to protect, and "install unknown apps" turned on.
- The user chose the Chrome install.

## Follow-up
- Two people saving a session at the same moment could both get the same session number. That's acceptable with 3 users.

## Excluded tags + payer buttons (2026-10-30, commit `78932e7`)
- **H** (paid from the joint bank account) and **R** (refunded) are tags that mean "handled, nobody owes it". `shares()` returns `{}` for them, as opposed to `None` for untagged. So they count as tagged (they leave the To-tag list and get saved with the session) but add nothing to J/B totals or the balance ledger.
- Totals shows H as its own series. R is left out because the money came back.
- H/R live in the row dropdown ("Not split") so the row stays J · B · T. A row shows an H or R button only once one is chosen (tap it to clear).
- Card payments have J / B / H buttons on the row. H stores `paid_by=joint`, which credits half to each person.
- Test gotcha: a refund appears twice in the CSV, once as the charge (taggable) and once as the credit (not taggable). Pick the row by sign as well as amount.

## Pay-back tracker ("money tree"), 2026-10-30
- A plan can have a `payoff` block: `{label, who, to, lines}`. Its lines are priced like plan lines and summed into a live `goal`.
- Direct payments between people (cash or bank transfer, not on the card) go in a new **`payoffs`** sheet tab: `payment_id, date, amount, note, added_by, added_at`. The tab is created on the first payment.
- **API:**
  - `POST /payoff` takes `{amount: 0 < x ≤ 100000, note ≤ 200}`. The date is set on the server in America/Chicago, since the server clock is UTC.
  - `DELETE /payoff/{32-hex id}` removes a payment. It deletes the sheet row with a `deleteDimension` request.
  - `/balance` attaches `payments` and `paid` to the plan.
- **UI:** an inline SVG tree on the plan card. Seed → sprout → tree (full size at 75%), with 12 coins that each stand for 1/12 of the goal. Paid off adds sparkles and fallen coins.
  - All motion is CSS transitions and keyframes, and is off under `prefers-reduced-motion`.
  - The previous progress is kept in memory, so adding a payment animates from the old state.
- **Gotchas:**
  - `.8 * 12 / 12` is `0.8000000000000002` in floating point, so the last coin's threshold came out a hair above 100%. It now uses an epsilon.
  - CSS transforms on SVG `<g>` need `transform-box: fill-box` to scale around their own center.
- The old plans JSON was saved locally before it was replaced. The settings `plans` key now holds a single plan.
- **Second theme: Minnesota wildflowers.** You switch between "Money tree" and "Wildflowers" on the card. The choice is saved per device in `localStorage` (`mb-payoff-theme`), wrapped in try/catch.
  - Twelve hand-drawn native species bloom one per 1/12 of the goal. The last is the state flower (showy lady's slipper), and monarchs appear when it's paid off.
  - The sky clears as it grows: a sunny gradient layer fades in over a gray one, the sun rises, and the gray clouds drift off.
- **Open-source art considered, not used:**
  - Fluent Emoji (MIT), Twemoji (CC-BY 4.0) and OpenMoji (CC-BY-SA 4.0) only have generic flowers, no Minnesota species.
  - Mixing their style with ours would look inconsistent.
  - Third-party SVGs can carry scripts, so they'd need vetting.
  - Hand-drawn inline SVG means no dependency and nothing to license.
- **Gotcha:** a plan line for a charge that is also tagged to a person counts it twice. Tag it **or** list it in the plan, not both.
