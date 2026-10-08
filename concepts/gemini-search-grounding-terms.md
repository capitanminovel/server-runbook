# Gemini "Grounding with Google Search": what Google's terms allow (researched 2026-10-08)

## What it is
Gemini can search Google while it answers (`tools: [{google_search: {}}]`). The answer is a **Grounded Result**; Google
also returns **Search Suggestions** (an HTML snippet of search chips) and **Links** (the source URLs and titles). It's a
paid feature: on Gemini 3.x, 5,000 searches/month free, then $14 per 1,000.

## Why it matters: the terms don't fit a stored-profile app
Source: https://ai.google.dev/gemini-api/terms, section "Grounding with Google Search" -> "Use Restrictions".
Exact sentences that rule out how dispo_menu would use it:
- "will only display the Grounded Results with the associated Search Suggestion(s) to the end user who submitted the
  prompt" -- our profiles are shown to all staff and on the customer menu, without Google's suggestions.
- "will not ... cache, frame, syndicate, resell, analyze, train on, or otherwise learn from Grounded Results" -- a
  strain profile is stored for good; storing is only allowed for up to 2 years for display evaluation, chat history,
  or a temporary resubmission.
- "it is a violation of these terms to use Grounding with Google Search to extract or collect one or more of these
  components for another purpose (for example, using programmatic or automated means to collect Links, using Links to
  build an index, or using Links to identify destination pages for crawling or scraping)" -- this is exactly the
  "Gemini finds pages, our code opens them to check" design on the `gemini-research` branch.
- "will not modify, or intersperse any other content with, the Grounded Results" -- handing them to Claude to
  rewrite into a profile is modifying.

Conclusion: **Gemini + Google Search can't be our research finder.** It suits a chat answer shown once to the person who
asked, not a database of profiles.

## What Gemini CAN do for us
- **Writer without tools.** A normal Gemini call (no search) is covered by the general Gemini API terms, so it can turn
  OUR research (pages our own code found and read) into a profile, at ~0.7c vs Claude's ~10c (3.8 Flash, launch price
  until 2026-12-31). Paid tier: Google doesn't train on prompts/responses.
- Gemini 3 supports a strict JSON schema (structured output), so the writer's answer has the same shape as Claude's.
- **URL context tool** (Gemini reads URLs you give it) is not part of the grounding terms, but don't use it to read
  sites that block our own reader (e.g. Leafly behind Cloudflare): that would get around a bot block through Google.

## What it does NOT change
- Our own finders (research sites, sitemaps, brand pages, Sweed, "Pages to read") stay as they are.
- The other-name search uses Anthropic's web search (Haiku). Anthropic publishes no storage/display rule like Google's
  (results come back to Claude only; we keep Claude's cited URLs and check them ourselves). Re-check if that changes.
- Other search APIs: Bing Search API retired (2025). Brave Search API ~$5 per 1,000, but storing results needs a
  separate, unpriced "storage rights" plan -- same problem.

## Key commands
```bash
# re-read the section if Google changes it
curl -sL https://ai.google.dev/gemini-api/terms | sed 's/<[^>]*>/ /g' | tr -s ' ' | grep -o 'Use Restrictions.\{0,2500\}'
```
