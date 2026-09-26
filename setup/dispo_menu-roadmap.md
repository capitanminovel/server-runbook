# dispo_menu — roadmap and open items (kept current)

## Needed, decided to do LATER
- **Onboarding plan for new dispensaries / new products** (decided 2026-09-25: "do later, just mark as needed").
  Question to settle: how to profile a whole menu (100-200 products) cheaply. Options noted, none chosen:
  1. a **shared strain library** across dispensaries (generate a strain's public facts once; per-dispensary data — links,
     sizes, lab data — stays separate; must respect the DB-per-owner isolation plan, so only public strain facts are shared);
  2. **Anthropic batch pricing** (50% off for non-urgent jobs) fed by the same free research bundles;
  3. generate the **first big batch inside Claude Code** (subscription instead of API cost; our code prepares each strain's
     research free, results are written to the DB) — suits the first dispensary, not self-service SaaS;
  4. a **cheaper model** for easy strains (Sonnet was thinner in an early test; re-test on the 17 research cases first).
  Current cost per strain 5-24c (a 150-product menu ~= $8-35). Keep testing a few brands with the saved test cases.
- ~~Sync from Live redesign / Flagged Queue~~ — **done 2026-09-25: both removed.** Refresh live menu + click-to-link replace them (see below).

## Other open items
- Cautions phrase list needs legal/compliance review (still a draft by Claude).
- `dev` / `main` branch split (everything is on `main` today); production checklist: prod `.env`, session secret, decide the
  Basic Auth wall, migrations on the prod DB, remove test data/demo login.
- Edibles product type (Grasslandz etc.); Marawanna / Redwood County Weed Co / brand websites still blank.
- Customer-facing live menu tile (reads profiles through the Sweed links); Product History / Menu Editor / Education tiles.
- Sold-out products: Sweed's public storefront drops them entirely; a fuller Sweed API (back-office key) would show them as
  inactive -- ask the Sweed rep (see incident notes).
- Deep-search button with spend tiers (idea from earlier; the Misc search covers the main need).

## Done 2026-09-25
- **Edit screen** on every strain card (text fields, comma lists, Cautions as the fixed approved checkboxes); each changed field is marked
  "edited" and a re-run of the AI (`redo`) leaves it alone (`strains.edited_fields`; verified with the AI stubbed).
- **Refresh live menu** (`POST /api/sync/refresh`, ~20-30s): fetches the menu once; updates each linked product's on/off-menu flag, sizes and
  prices; strain = Active if any linked product is on the menu else Not active; lists menu products with no profile (43 on the menu, 38 with no
  profile at test time). Never creates strains or flags, never overwrites text. An empty pull changes nothing (protects against a bad fetch).
- **Flagged Queue removed entirely** (page, route, `/api/flags`, `sync_flags` table + 3 enums, `dispensaries.sync_review_mode` + enum) and the **old
  Sync from Live** (`run_sync.py`, `POST /api/sync/`, its button, its nginx location) -- they only existed to feed the flag queue. Migration
  `b8d3f0a6c512` is reversible (round-trip tested); the table was empty. The Product History tile is still a placeholder reading `sync_log`
  (nothing writes to it now; Refresh could log snapshots when that tile is built).
- Link to live menu is the replacement for the old sync's match suggestions: the app finds candidate products (name, type, brand, sizes) and one
  click connects them (never silent auto-link, per the project rule).

## Done 2026-09-25 (later)
- Roles (admin / employee, server-enforced), Team page, shared employee login supported; Education tile (sections, trainings, files,
  video, viewer, drag to rearrange). See `setup/dispo_menu-education-and-roles.md`.
- Security: API runs as unprivileged `dispo` user, sandboxed (`concepts/least-privilege-services.md`); Secure 12 h cookie;
  Starlette vulnerabilities fixed.

## Next / open (Education)
- Individual employee logins (planned later; Team page already supports it) and who-read-what tracking (uses the Need to Read tag).
- PDF.js viewer if staff use phones; storage move (Spaces / Drive) when disk gets tight; server RAM (~1 GB, heavy swap) — a 2 GB
  droplet would give headroom; database user hardening (the app's DB role still owns its whole database).

## 2026-09-26
- **Brand suggestions from the live menu**: "+ Add new brand" lists menu brands not in your list (exact spelling) and brands spelled
  differently ("Use menu spelling"). The list is saved on the dispensary (`dispensaries.menu_brands`) by Refresh live menu and by the
  lookup itself; the form reads the saved list instantly and only reads the live menu (~12-30 s) when it is older than 6 h.
- **Refresh live menu** now also suggests links (menu products matching an existing strain, one click) and names what went off the menu.
- **Code map**: `Dispo_menu/docs/CODE_MAP.md` — where everything lives and where to change it (linked from CLAUDE.md).
- **Next: scheduled Refresh (cron)** — run Refresh live menu on a timer so statuses, sizes, prices and the brand list stay current
  without anyone clicking. Watch memory: each run starts a headless browser (~150-250 MB) on a ~1 GB droplet.

## Sweed official API + live-menu CPU (2026-09-26)
- **Measured:** one Playwright menu read takes 15–20 s at 100% of the single CPU. Blocking images and fonts doesn't help, because the cost is Chrome plus the store's JavaScript. Hourly refreshes are fine for up to about 30–50 dispensaries if they run one at a time, staggered, at low priority (`Nice`/`CPUWeight`), during store hours only.
- **Sweed launched a Public API (Aug 2026).** Key comes from the retailer's Sweed account manager and is bound to specific stores. `GET /v2/stores/{id}/products` returns per-variant stock and prices plus `compounds` (THC, CBD, terpenes); the detail endpoint adds COA `documents`. Rate-limited, so cache server-side. Webhooks are "coming soon". **Price not published.** Docs: https://api-demo.sweedpos.com/docs/#ecom-api-v2
- Next: ask Legit's Sweed rep (see the questions in chat / below). If we get a key, swap `sweed_client._fetch_live` to the API and keep Playwright as the fallback.

## Live menu now read without a browser (2026-09-26)
- Tested: Sweed's storefront endpoints (`/_api/Products/GetProductList`, `GetExtendedLabdata`) answer a plain HTTPS POST with just the `storeid` header. No cookies, no Chrome. This is the same request the public menu page makes for every visitor, and it isn't behind a bot check.
- `sweed_client.py` now does that, with an honest User-Agent (`dispo_menu/1.0 …`). **Full menu: ~1 s and ~0.1 s CPU, down from 15–20 s at 100% CPU.** Memory per read went from ~200 MB to a few MB.
- **Backup:** if the first direct call fails, it logs `Sweed direct call failed … falling back to the headless browser` and uses the old Chrome path. If that line shows up in `journalctl -u dispo-menu-api`, Sweed changed something.
- These endpoints are internal and undocumented and can change without notice. The official Public API key (ask Legit's Sweed rep) is still the long-term route, but it's no longer urgent.
- Effect: hourly scheduled refreshes are now cheap even for many dispensaries. The CPU/queue concerns above mostly go away. Playwright stays installed only for the backup.
- Tests after deploy: research_check 16 pass / 1 warn (MAC Stomper menu changed) / 0 fail; ui_check 29/29; no browser fallbacks.

## Server check + updates (2026-09-26)
- **Tests after all changes (regression):** jobs_check 13/13, schedule_check 17/17, education_check 23/23, ui_check 29/29,
  research_check 16 pass / 1 warn (MAC Stomper menu listing changed) / 0 fail. smoke_paid not run (costs Claude credits; nothing in the AI call changed).
- **Packages:** Python 0 known vulnerabilities (pip-audit), admin site 0 (npm audit). OS: 44 updates installed;
  held back `fwupd` (major version jump, irrelevant on a VM) and `xvfb`/`xserver-common` (Ubuntu phased rollout — will arrive by themselves).
- **Reboot needed** for kernel 6.8.0-138 → 142 and to restart services using updated libraries (postgres, ssh, journald, cron, dbus).
  All services on the droplet (dispo_menu, money_buddy, capitans_terps, Pi-hole) go down ~1–2 min. Steps:
  ```bash
  reboot
  # reconnect after ~1 min, then:
  uname -r                                   # 6.8.0-142-generic
  systemctl --failed                         # should list nothing
  systemctl is-active nginx postgresql@16-main dispo-menu-api pihole-FTL money-buddy capitans-terps fail2ban
  systemctl list-timers 'dispo-menu-*'       # all three scheduled
  curl -s https://dispo-api.dev.withcapitan.com/health
  ```
  tmux sessions don't survive a reboot: afterwards `tmux new -A -s claude`, then `claude --continue`.
- **Healthy:** no failed units; firewall default deny, only 22/80/443 open to all, Pi-hole 53/8080/8443 only to the trusted IP;
  SSH keys only (password auth off); fail2ban jails sshd + nginx-botsearch + nginx-probe; certs valid to 2026-10-31 (certbot timer on);
  disk 62 %; unattended security upgrades on.
- **Found, awaiting OK:**
  - **CUPS printing service** (snap `cups`) runs as root listening on 0.0.0.0:631. Blocked by the firewall's default deny,
    but a server has nothing to print → `snap remove cups`.
  - **`PermitRootLogin yes`** in `/etc/ssh/sshd_config`. Passwords are already off, so it's effectively key-only; setting
    `prohibit-password` makes that explicit if password auth is ever re-enabled by mistake. (Longer term: a non-root sudo user.)
    SSH changes risk a lockout: test `sshd -t`, keep the current session open, and try a second login before closing it.
  - **Memory:** 961 MB total, ~300 MB available, 510 MB swap in use — mostly Claude Code (~400 MB). Moving Claude to the
    laptop or a 2 GB droplet is still the main headroom fix.
  - Turn on **DigitalOcean droplet backups** (off-server copies; on-server nightly backups only cover mistakes).
  - Outside uptime monitor (UptimeRobot) on `/health` — the only thing that can tell you the whole droplet is down.
