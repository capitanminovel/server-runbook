# Minnesota cannabis advertising rules (as they apply to our menu)

Researched 2026-09-29 for the dispo_menu customer menu and kiosk. **Not legal advice**: a compliance lawyer should
review before the menu is public.

## What it is
Minnesota regulates cannabis **advertising** through **Minnesota Statutes 342.64** plus the Office of Cannabis
Management's (OCM) **Guidance Memo GM-2025-07** (updated Aug. 7, 2026). The rules chapter (Minnesota Rules 9810) covers
licensing, packaging, labeling and retail operations, but has **no advertising section**.

## Why it matters / how it works
- The OCM defines an advertisement as "any written or oral statement, illustration, or depiction that is intended to
  promote sales", including the internet. An online menu with prices and descriptions very likely counts, so we treat
  our menu and kiosk as advertising. (Fixed outdoor signs on the building are the one listed exception.)
- **342.64 subd. 1** — no ad may: (1) be false or misleading; (2) make **unverified health or therapeutic claims**;
  (3) promote overconsumption; (4) show anyone under 21 consuming; (5) use images likely to appeal to under-21s
  (cartoons, toys, animals, children); (6) show alcohol; (7) **lack the OCM's warning**.
- Subd. 4: no unsolicited pop-up ads online. Subd. 5: **age verification** before direct/individual advertising.
- **Required warning, verbatim** (cannabis):
  > Warning: Cannabis products are not for use by anyone under the age of 21. Cannabis use may cause drowsiness,
  > affect focus, reaction time, and decision-making. These products are not evaluated or approved by the FDA. Pregnant
  > people should avoid cannabis due to the risk of low birth weight, premature birth, stillbirth, and harm to fetal
  > brain development.

  Hemp products with THC (lower-potency hemp edibles) have their own wording (in `LegalNotice.tsx` as `HEMP_WARNING`).

## How it's set up in dispo_menu
- `apps/customer-menu/src/components/LegalNotice.tsx` holds ALL legal text: `OCM_WARNING` (verbatim), `HEMP_WARNING`,
  and the information-only notice (`ABOUT`). Shown on: the age check, the web menu footer, every strain page, the kiosk
  (a bottom bar that's always on screen) and the kiosk slideshow.
- **Reported uses** (the `therapeutic` field) is hidden on the menu: `SHOW_REPORTED_USES = False` in
  `apps/api/app/public_menu.py`. It's still generated for staff/admin reference.
- The Generator's instructions forbid health/therapeutic claims in `description` and Misc (customer-facing).
- **Cautions** are hidden everywhere (admin cards too) until the legal review.
- The 21+ age check was already in place (`AgeGate.tsx`); it isn't a pop-up ad.
- Demo menu: legitdemo-menu.withcapitan.com, behind a password and `X-Robots-Tag: noindex`.

## What it does NOT do
- It doesn't make the menu compliant by itself: descriptions and facts are AI-written from research, so new ones should
  still be spot-checked for health wording ("relief", "helps with", conditions).
- The text scan used to find "stress relief" is a one-off; there's no automatic filter.
- Whether the menu must show the retailer's **license number** or business name wasn't found in 342.64, GM-2025-07 or
  chapter 9810 — ask the lawyer.
- If **hemp/THC edibles** are ever listed, the hemp warning must be shown too (not wired up yet).

## Key commands
```bash
# find health-claim wording in customer-facing text (run per database)
sudo -u postgres psql -d dispo_menu_demo -c "select id,name,description from strains where status='active'
  and description ~* '(relief|treat|helps with|pain|insomnia|anxiety|depress|nausea|ptsd|medic|therap)'"
```

Sources: [MN Stat. 342.64](https://www.revisor.mn.gov/statutes/cite/342.64) ·
[OCM GM-2025-07](https://mn.gov/ocm/businesses/guidance-memos.jsp?id=1202-709836) ·
[MN Rules ch. 9810](https://www.revisor.mn.gov/rules/9810/full)
