# Cannabis terpenes, strain names and effects: what the evidence supports (2026-10-14)

## What it is
Terpenes are the aroma compounds in cannabis (limonene = citrus, myrcene = earthy/musky, caryophyllene = pepper).
Lab results (COA, Sweed's per-product lab data) measure how much of each a product has. Menus often turn those
numbers into feelings ("myrcene = relaxing"). This note records how solid that is, and what dispo_menu does about it.

## Why it matters / how it works
- **Aroma: solid.** Terpenes ARE the smell; a product high in limonene smells citrusy. Safe to say.
- **Effects from terpenes: weak.** Mostly animal studies. The best human result: vaporized d-limonene reduced THC's
  anxiety/paranoia in 20 adults (Johns Hopkins, Spindle et al. 2024, Drug and Alcohol Dependence); limonene alone
  did nothing. No good human evidence that myrcene sedates or that caryophyllene calms at menu-level amounts.
- **Strain names: unreliable.** Same-name samples from different growers are often genetically different (only 4 of
  30 strains consistent; J Cannabis Research 2019). Strain-database "effects" are reviews of the NAME across many
  growers, not of this grower's product.
- **Indica/Sativa labels: don't match chemistry** (Smith et al. 2022, PLOS ONE, 6 US states). Most THC products
  fall into 3 terpene groups: caryophyllene-limonene, myrcene-pinene, terpinolene-myrcene.

## How it's set up on this server (dispo_menu)
- Menu refresh keeps each linked product's own lab result (every terpene) in `strain_sweed_links.variants[].lab`;
  the customer menu shows THC, total terpenes, top 5 per product (`app/sync/refresh.py`, `app/public_menu.py`).
- Measured on our store 2026-10-14 (50 products, 26 with terpene data): caryophyllene is in the top 3 of ALL 26 and
  leads 15. That's why a terpene->feel table said "Calm" for 22 of 26 -- it described cannabis in general, not the
  product. Comparing each product with the menu's typical amount separates them well (Glitter Bomb, Grape Cream Cake,
  Octane Mintz = myrcene; Gorilla Zkittlez, Magic Marker, Zoap = limonene + pinene; Lit OG, Runtz, Guava = linalool).
- Profile "effects" come from strain databases (AllBud, CannMenus...) and the store's tags: reports about the name.

## What it does NOT do
- Terpene numbers don't predict how a product will feel for a given person; THC dose, tolerance and setting matter more.
- 48% of our products have no terpene data in the store; those can only show the profile.
- The store's own effect tags include health claims ("Stress Relief") -- never show them unfiltered (MN 342.64).

## Key commands
    cd /root/Dispo_menu/apps/api && venv/bin/python <scratch>/menu_terps.py   # re-measure the menu (free, ~70 store reads)
