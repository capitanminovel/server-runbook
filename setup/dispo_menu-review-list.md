# dispo_menu: things still to review (as of 2026-09-28)

Tick these off (or move them into `dispo_menu-roadmap.md`) as they're decided.

## Content — your call / needs a person
- [ ] **Admin Guide in Education is the old PDF** (training #54, dev + demo). Open the new
      `docs/guides/Dispo-Tool-Admin-Guide.docx`, save as PDF, **Replace file**. Consider adding the Employee Guide too.
- [ ] **Papaya Fuel lineage looks like a typo**: store says "Critical #13 x Ice #2"; the classic *Papaya* (Nirvana)
      is **Citral** #13 x Ice #2. Confirm with Unbound, fix the lineage in Edit, then Redo (it may be "Papaya").
- [ ] **Swedish OG** (Uffda OG x Peach Bio): nothing online. Ask Tasty Gems for tasting notes -> "specifically included".
- [ ] **Fight Club**: no lineage anywhere. Ask Campfire Cannabis for the cross.
- [ ] **Golden Goat**: research sites say sativa, the store says indica (the store label is shown). Confirm.
- [ ] **Spot-check the 1–2 page profiles**: Hyperion F1, Jacked Up Boof, Block Party, Halle Berry, Wilfunk.
- [x] **Test duplicates deleted (2026-09-28):** demo #183–185; dev #186 (Double Sour Grape copy) and #190 (Tropical
      Fresa, wrong "Trop Strawberry" match). #190 had taken the live-menu link from the original #152 Tropic Fresa, so
      the link was moved back to #152 first. Rows backed up in `/root/backups/test-duplicates-20260928/`.
- [ ] **JointCommerce** is often AI-written filler (Burger Breath "grilled meat", Wilfunk "extrapolated"). Consider
      removing it in the Generator's Sources tab, or keep it as a last resort (the lineage check already guards names).

## Legal / going public
- [ ] **Cautions list** is a first draft — needs legal review.
- [ ] **Customer menu & kiosk wording** ("Reported uses", disclaimers) — legal review before they go public.
- [ ] **Domain** for the public menu (ideas: yourdispotool / dispotool / urdispotool).

## Features — decisions
- [ ] **Lab reports (COA)**: dev only. Move to the demo? (Scanned COAs aren't read; would need OCR or an AI read.)
- [ ] **Lineage check for ALL research pages**, not just other-name ones (would catch Papaya Fuel/Terp Poison-type
      mix-ups automatically).
- [ ] **Name search: 1 web search instead of 2** (~half the cost); test whether it still finds names.
- [ ] **Compact terpene guide**: not adopted (accuracy). Retest later with a "dominant terpenes only" rule.
- [ ] **Auto-link** new profiles to live-menu products (today it's an admin click by design).
- [ ] **Demo limits**: 10 AI generations, 20 name searches — right numbers for the team?
- [ ] **Individual logins** for the team (Team page) instead of one shared admin login.
- [ ] **Store hours** for the 5-minute refresh (08:00–22:00 America/Chicago) — confirm.

## Server / ops
- [ ] **Email alerts** aren't sent yet (needs a Gmail app password).
- [ ] **RAM**: 1 GB droplet running dev + demo leans on swap (~700 MB). 2 GB droplet recommended.
- [ ] **Off-server backups** (DigitalOcean backups) and an **uptime monitor**.
- [ ] **Server reboot** for the pending kernel update.
- [ ] Waiting for your OK: remove the **CUPS snap**; set **PermitRootLogin prohibit-password**.
- [ ] **Sweed Public API key** from Legit's Sweed rep (stock, terpenes, COA links).
- [ ] **Separate Anthropic keys** for dev and demo, so spend can be told apart (same key today).
