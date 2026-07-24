# Grape Nebula: AI research generated wrong genetics lineage

## What happened
The "Generate Research" feature in capitans_terps produced a genetics/lineage
writeup for Grape Nebula claiming it was a cross of "Grape Pie x Zkittlez/Purple
Punch." The actual known cross, recorded in the strain's notes when it was
added, is `Dulce De Uva x Starburst F2`. Claude fabricated a plausible-sounding
but wrong lineage.

## Why
`routers/research.py` builds its prompt to Claude using `strain["lineage"]` as
a grounding hint ("Known lineage: X.") if that DB column is set. But the Add
Strain form (`static/index.html`) never had a Lineage input field — only Name,
Breeder, Status, and Notes. So when Grape Nebula was added, the real cross went
into free-text Notes instead of the `lineage` column, which stayed empty.

With no lineage hint in the prompt, Claude had only the strain name and
breeder to go on. It pattern-matched "Grape Nebula" against real-world strains
with that name from its training data and generated a specific, confident,
and wrong genetic lineage — a hallucination masked by house style (research
was 3-4 sentences of technical, textbook-sounding detail, same as every other
correctly-grounded entry).

The `lineage` column and backend support (`routers/strains.py` `NewStrain`
model, insert query) already existed — only the UI to populate it was missing.
Schema was ahead of the form.

## Fix
```bash
# Set the real lineage on the strain, clear the bad cached research row
python3 -c "
import sqlite3
conn = sqlite3.connect('data/capitan.db')
conn.execute('UPDATE strains SET lineage = ? WHERE id = ?',
             ('Dulce De Uva x Starburst F2', 'grape-nebula'))
conn.execute('DELETE FROM research WHERE strain_id = ?', ('grape-nebula',))
conn.commit()
"
# Regenerate through the live app so the prompt now includes the lineage hint
curl -s -X POST http://127.0.0.1:8001/api/research/grape-nebula
```
Also added a Lineage input to the Add Strain form (`static/index.html`) and
wired it into `submitAddStrain()` (`static/js/strains.js`) so future strains'
cross info lands in the field the research prompt actually reads, not in
free-text Notes.

Audited all other strains — grape-nebula was the only one with an empty
`lineage` column and cross info stranded in notes. No other strains affected.

## What I learned
- When an LLM is asked to be specific about something it has no real data
  for, it doesn't say "I don't know" — it pattern-matches to plausible
  training-data lookalikes and states it with full confidence. The output
  reads identically to a properly-grounded answer; only the content is wrong.
- A DB column existing and being wired into an insert query doesn't mean data
  is actually landing there — check the *form* that's the entry point, not
  just the schema and backend model.
- Grounding hints only help if the UI actually captures the data they depend
  on. Worth spot-checking any "if we have field X, use it as an AI hint"
  pattern for whether X is actually reachable from the UI.

## Follow-up
None — form fix is in place, other strains audited clean.
