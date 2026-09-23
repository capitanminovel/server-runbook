# dispo_menu — Anthropic credit balance exhausted mid-testing (shared key)

**Date:** 2026-09-23

## What happened
While benchmarking the Strain Generator (three parallel Opus 5 calls with web_search + web_fetch), two
of the three failed with `400 invalid_request_error: Your credit balance is too low to access the
Anthropic API`. Every AI call from dispo_menu now fails until credits are added. A 4-minute 504 on a
real generate request just before that may be related or may be plain slowness — not determined.

## Why
dispo_menu's `CLAUDE_API_KEY` is the same key as `/opt/legit-buddy-private`'s `ANTHROPIC_API_KEY`
(same account, decided 2026-08-25). One balance covers both apps, so this likely also affects
legit-buddy-private's AI enrichment. Generation with web search + full page fetches on an Opus-tier
model is much more expensive per call than the original no-tools version, and my benchmark ran
three of them at once. I can't tell from here whether the balance was already low.

## Fix
Not fixable from the server — needs credits added under Plans & Billing in the Anthropic console.
Code-side: generator routes (`app/routes/strains.py::_generate`) now turn Anthropic API errors into
readable messages (503 "out of credits", 502 otherwise) instead of a raw 500. Verified live.

## What I learned
- A key shared across two apps means a spike in one can silently take down the other; a separate key
  (or at least a spend cap/alert on the account) would have made this visible earlier.
- Benchmarking paid, tool-using calls in parallel is expensive — run one, not three.
- `effort: "low"` did NOT speed things up usefully: it returned every field empty after 4 searches,
  zero page fetches. Don't use it for this call.

## Follow-up
- [ ] Add credits, then re-verify the new generator end to end: Cherry Lady Slipper (Trailhead has a
      saved site) should fill `supplier_description` and cite the page; Blue Dream should cite
      AllBud/Leafly and fill `prevalence_type` and `interesting_facts`.
- [ ] Generation time is unmeasured with the larger budget (6 searches / 5 fetches). The generate
      route's nginx timeout was raised to 240s; if real runs are still slower than that, cut the
      budget or move to a background job with polling instead of one long request.
- [ ] Consider a separate key for dispo_menu and a spend alert on the account.

## Update (same day): credits restored, cost cut
Credits re-added; live Cherry Lady Slipper generation verified against the real Trailhead pages (a first
run stated an inference as fact — "name is a nod to MN's state flower" — so `interesting_facts` was
tightened to facts a page states directly). Each Opus 5 generation with search + full page reads costs
roughly $0.50–1+, which the user objected to. Changes: the Strain Generator model is now
`STRAIN_GENERATOR_MODEL` in `apps/api/.env` (default `claude-sonnet-5`; set `claude-opus-5` and restart
`dispo-menu-api` to switch back) and `web_fetch` is capped at 8000 tokens per page. Per-generation token
usage + model are logged: `journalctl -u dispo-menu-api | grep "strain generation"`.
- [ ] Not yet re-tested on Sonnet 5 — check the first real run's token log and that facts still ground.
- [ ] Option not built: split "Generate" (cheap, model knowledge only) from an on-demand "Research sources" button.
