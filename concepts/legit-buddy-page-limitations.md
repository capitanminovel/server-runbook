# Legit Buddy / Dinky Buddy — Page Limitations to Communicate to Owner

**Date:** 2026-06-28

---

## 1. Mood Scores Are Only as Good as the COA Data Entered in Sweed

The app scores every product for moods (Wind Down, Pain & Body, Lift Up, etc.) based on terpene data pulled from Sweed POS. **That data is only as accurate as whoever entered it.**

Problems that silently break the scores:
- Terpenes left blank → app falls back to Claude's general strain knowledge (generic, not batch-specific)
- Terpenes copied from marketing copy or a website instead of the actual lab COA → scores reflect brand claims, not the real product
- Terpenes entered once and never updated when a new batch arrives → scores reflect a different batch's chemistry

**What accurate looks like:** Staff receives a product, opens the COA PDF from the brand, and enters the terpenes in Sweed in the exact order listed on the lab report (order matters — the app weights position #1 most heavily).

**Recommendation for owner:** Make accurate COA entry a product intake requirement. The mood scores are a differentiator vs. other dispensary tools — but only if the underlying data is real.

---

## 2. Schedule Delay — Menu Updates Are Not Instant

Both `legit-buddy-api` (GitHub Pages) and `dinky-buddy-api` (GitHub Pages) run on GitHub's free tier. Scheduled scrapes are queued and delayed by **1–4 hours** past the scheduled time depending on GitHub runner load.

Current schedule targets (actual run times):
- ~7:00 AM CST
- ~11:00 AM CST
- ~3:00 PM CST
- ~6:00 PM CST
- ~8:00 PM CST

If inventory changes in Sweed at 9 AM, the menu may not reflect it until 11 AM or later.

**legit-buddy-private** (the upgraded version on the DigitalOcean droplet) runs cron on the server and hits exact times — this delay does not apply to it.

---

## 3. No Manual Refresh Button (Public Sites)

Attempts were made to add a one-click refresh button to the GitHub Pages sites so staff could trigger an immediate scrape. This is not possible for public GitHub repos — GitHub automatically revokes any token embedded in public HTML. The button was removed.

**Workaround:** Go to GitHub → Actions → Daily Menu Scrape → Run workflow to trigger a manual scrape outside the schedule.

**Long-term fix:** Move dinky-buddy-api to the same droplet setup as legit-buddy-private. This would enable a token-protected `/api/menu/refresh` endpoint callable from the page, exact schedule timing, and real-time data.

---

## 4. "New Arrivals" and "Sold Out" Windows Are Time-Based

- **New in Last 3 Days** — shows products first seen within 3 days. If a product was in Sweed before the scraper first ran, it won't appear as new even if it's new to staff.
- **Sold Out — Last 2 Days** — shows products that dropped out of inventory within 2 days. Products gone longer than that disappear from the sold section silently.

These windows are hardcoded in `build_preview.py` / the pipeline and can be changed if needed.

---

## 5. API Keys and Repository Are Currently Owned and Managed by Jose

This tool runs on Jose's GitHub account and uses his Anthropic API key. That means:

- **GitHub repo:** The code and the automation (scrapes, deploys) live under Jose's GitHub account. If Jose stops managing it, the pipeline stops.
- **Anthropic API key:** The AI that generates strain profiles (lineage, therapeutic uses, mood ratings) runs on Jose's key. If usage grows or the key is revoked, enrichment stops.
- **No handoff path yet:** There is no self-service way for the store to take over — it requires someone technical to manage.

**What this means going forward for a new store:**

The store owner has two options:
1. **Jose manages everything** — Jose owns the repo and key, charges a monthly fee that covers management + API costs. Store gets a working tool, no technical burden.
2. **Store owns their own accounts** — Store creates their own GitHub account and Anthropic API key (both free to set up). Jose builds and hands off, but store needs someone internal or retains Jose for ongoing changes.

**Recommendation:** Option 1 is cleaner for a non-technical owner. Jose manages the infrastructure, bills monthly, and is the single point of contact. If they ever want out, the code and data can be exported.
