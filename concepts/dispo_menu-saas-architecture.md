# dispo_menu as a SaaS: architecture review and roadmap (2026-10-14)

## What it is
A check of how dispo_menu is built against how successful SaaS products are built, and what to change -- in order --
before a second dispensary (customer) signs up. Written after the menu/terpene, New & Sold out, courses and games work.

## How it's built today (and why that's right for now)
- **Modular monolith**: one FastAPI app + one Postgres DB, one route file and one page per tile, background jobs as
  separate processes (systemd timers). This is what Shopify chose over microservices: strict boundaries between
  features, but one deploy, no network calls between services, easy refactors
  (shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity).
  -> Keep it. Microservices would add cost and failure points with no benefit at our size.
- **Multi-tenant "pool" model**: every tenant-owned row carries `dispensary_id` (AWS SaaS Lens calls this the pool model:
  cheapest, least isolation; docs.aws.amazon.com/wellarchitected/latest/saas-lens/silo-pool-and-bridge-models.html).
  The earlier decision (memory: DB-per-owner later) = the "silo" tier for owners who need it -> AWS "tier-based isolation".
- **Research pipeline**: code gathers (free), AI only writes, code checks, admin decides. Every external thing sits behind
  one module (Sweed: `sync/sweed_client.py`; AI: `ai/client.py`; web: `ai/page_fetcher.py` with the SSRF guard).
- **Content as data**: courses (`courses/content/*.ts`) and games (`games/`) are registries -- add a file + one line.
- **Safety by construction**: public menu fields are a whitelist; `require_admin` on every write; pinned dependencies;
  open-source vetting checklist in the project skill.

## What to change before customer #2 (in order)
1. **Per-tenant store config** -- BLOCKER. Sweed store id/domain/category ids are hard-coded for Legit south-metro
   (`sync/sweed_client.py` STORE_*). Move them to a `dispensary_settings` table (or columns on `dispensaries`).
2. **POS adapter interface** -- wrap Sweed behind a small `PosProvider` (list products, lab data, product URL) so another
   POS (Dutchie, Treez, Blaze...) is a new adapter, not edits everywhere. Same idea as the AI client boundary.
3. **Row-Level Security in Postgres as a second lock** -- today isolation depends on every query remembering
   `dispensary_id`. RLS policies (`USING dispensary_id = current_setting('app.dispensary_id')::int`) make a forgotten
   filter return nothing instead of another store's data. Defense in depth; costs little.
4. **Per-tenant settings, plans and limits** -- AI_GENERATION_LIMIT, feature flags (COA files, name search), branding
   (logo, colors, product name) live in env vars / code today. Put them per tenant so plans ("Basic" / "Pro") are data.
5. **Shared vs tenant data** -- strain research (pages read, lineage) is the same for every store; profiles, links,
   prices are per store. A shared research cache would cut cost/time per tenant and be gentler on research sites. Keep
   the split explicit: `global_*` tables never hold tenant data.
6. **State-specific content** -- the "Customers & Minnesota Rules" course is Minnesota-only. Tag courses with a
   jurisdiction and show the tenant's state; other states need their own rules course (reviewed by a person).
7. **Operations** -- CI that runs the free checks on every push (GitHub Actions); error tracking with the tenant id on
   every log line (self-hostable GlitchTip/Sentry); per-tenant export + delete (customers will ask; privacy laws require
   it); an admin audit log (who changed/linked/deleted what).
8. **Jobs at scale** -- systemd timers are fine for a few tenants. With many: one scheduler process looping tenants with
   per-tenant backoff (already how menu_refresh works), then a Postgres-backed queue before ever adding Redis.

## Coding rules that keep it clean (already followed -- keep doing it)
- One file per feature; registries for content (courses, games) and external systems (adapters).
- Every outside system behind one module; nothing else imports the vendor library.
- Free tests next to each feature (`scripts/*_check.py`); paid tests only with a stated cost.
- Dependencies: vet, pin, audit, prefer a small own implementation over a heavy package (2026-10-14: Trafilatura,
  RapidFuzz, extruct, Cannlytics evaluated -- none added; a 20-line junk-line filter beat Trafilatura on our pages).
- Migrations for every schema change; whitelists for anything public.

## What it does NOT need (yet)
Microservices, Kubernetes, Redis, a separate search engine, GraphQL. Revisit only when a measured problem asks for it.

## Key references
- AWS SaaS Lens, silo/pool/bridge + tier-based isolation: docs.aws.amazon.com/wellarchitected/latest/saas-lens/
- Shopify modular monolith: shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity
- PostgreSQL Row Level Security: postgresql.org/docs/current/ddl-rowsecurity.html
