# Capitans Terps: strain type reclassification via real genetics audit

## What happened
Follow-up to [[2026-07-24-grape-nebula-hallucinated-lineage]]. After expanding
the strain `type` field from a 3-value system (Hybrid/Indica/Sativa) to a
7-point spectrum, the initial reclassification of all 10 strains was based on
reading each strain's own AI-generated `effects` research text. That's the
same generation pipeline that hallucinated Grape Nebula's lineage, so it was
cross-checked against real strain-genetics data before trusting it further.

The audit found the AI effects text got 3 of the 10 wrong:

- **pasture-bedtime**: text said "leaning heavily indica" → classified Indica
  Dominant. Real data: its Runtz parent is a genuinely balanced 50/50 hybrid,
  not indica-heavy. Corrected to Indica Leaning Hybrid.
- **spicy-guava**: text emphasized cerebral/energetic character, explicitly
  contrasted itself against "indica-dominant cultivars" → classified Sativa
  Leaning Hybrid. Real data: its RS11 parent is formally categorized
  indica-dominant (70/30) by most strain databases, not sativa. Corrected to
  Hybrid.
- **the-krux-16**: text called out "Sativa-leaning influence of ... Green
  Crack genetics" → classified Sativa Leaning Hybrid. Real data: Green Crack
  is a diluted great-grandparent; three closer, confirmed indica-dominant
  ancestors (Grateful Breath 70/30, Tres Dawg 60/40, Forum Cut GSC 60/40)
  outweigh it. Corrected to Indica Leaning Hybrid.

Two more were corrected in the other direction — the generic "Hybrid"/"Indica"
labels undersold how lopsided the real genetics are:
- **blueberry-pie**: was plain "Indica" pre-spectrum, then "Indica Leaning
  Hybrid." All three real ancestors (Triangle Kush 85/15, Sunset
  Sherbet-family Gelato #41 ~60/40, Georgia Pie ~60-70/30-40) are
  solidly indica. Corrected to Indica Dominant.
- **mind-flayer-s1**: generic "Hybrid." Its Permanent Marker parent is
  confirmed 70/30 indica-dominant. Corrected to Indica Leaning Hybrid.

## Why
Same root mechanism as the Grape Nebula lineage bug: the research-generation
prompt (`routers/research.py`) asks Claude for a 3-4 sentence effects
narrative, and a narrative has to pick a throughline. When one ancestor has a
recognizable name (Green Crack) or the strain name itself suggests a vibe
("Spicy Guava" sounds bright/energetic, "Pasture Bedtime" sounds sedating),
the model leans on that as the organizing idea even when the actual pedigree
weight points the other way. It's not fabricating facts this time — Green
Crack really is in The Krux #16's lineage — it's misweighting real facts
into a narrative that doesn't match the aggregate genetics.

## Fix
Cross-referenced each strain's named parent genetics (not the app's own
generated text) against real strain databases (AllBud, Leafly, etc. via web
search) and derived indica/sativa percentages for well-documented ancestors,
then estimated each capitans_terps strain's overall lean from its full
pedigree rather than from the AI narrative alone. Applied corrected `type`
values directly via SQL:
```python
updates = {
    'blueberry-pie':   'Indica Dominant',
    'mind-flayer-s1':  'Indica Leaning Hybrid',
    'pasture-bedtime': 'Indica Leaning Hybrid',
    'spicy-guava':     'Hybrid',
    'the-krux-16':     'Indica Leaning Hybrid',
}
```

## What I learned
- A generated narrative summary (research "effects" text) is a worse source
  for a structured classification than deriving it from the underlying
  structured data (parent lineage) directly. Ask "what does the pedigree
  actually average to" before trusting prose that was optimized to read well,
  not to be quantitatively accurate.
- Famous/recognizable ancestor names get overweighted in AI-generated
  narratives relative to their actual generational dilution — worth
  discounting distant, name-recognizable ancestors and weighting the closest
  generation more heavily when sanity-checking this kind of content.
- Real-world indica/sativa ratios for the same strain vary meaningfully by
  source/breeder (RS11 ranged from "70/30 indica" to "mostly sativa"
  depending on the site) — treat any single-source ratio as directional, not
  exact, and prefer the majority/more-authoritative-sounding classification
  when sources disagree.

## Addendum: sub-lineage text errors
While cross-checking, also found two confirmed factual errors in the
`genetics_lineage` prose itself (not just the type field) — same failure
mode as the original Grape Nebula bug, where a trendier/more-famous strain
name got substituted for the real, more obscure parent:

- **blueberry-pie**: claimed "Georgia Pie descends from Gelatti x ...
  Gushers." Real data (multiple consistent sources): Georgia Pie is
  Gelatti x **Kush Mints**. "Gushers" doesn't factor into Georgia Pie at all
  — likely pulled in because Blueberry Pie's *other* parent is "Blueberry
  Gushers," creating a false shared-Gushers narrative thread.
- **strawn-johnson**: claimed Strawnana descends from "Banana Kush and
  Bubblegum Gelato." Real data: Strawnana is Banana Kush x **Strawberry
  Bubblegum** (Serious Seeds) — no Gelato involved. Likely substituted
  because Gelato is an extremely common modern parent strain and pattern-
  matched in over the less-famous Strawberry Bubblegum.

Patched both sentences directly in the DB with verified facts rather than
regenerating (regenerating risked introducing a *new* unverified guess in
place of a verified one).

## Follow-up
None — all 10 strains now reflect pedigree-based type classification, and
the two confirmed sub-lineage text errors are corrected.
