[Русский](./architecture.ru.md) · **English**

# Architecture

Public architectural description — principles and structure only. No
client data, source code, or proprietary detail.

## Layers

1. **Git — source of truth.** Everything that defines the system is a
   file in the repo: pipeline scripts, niche DNA (taxonomies), client
   master taxonomies, branch geo configs, templates, DB migrations.
   Why: git log is a built-in audit trail; rollback is free; routine
   edits need no database; cloning the repo provisions the whole system.

2. **Pipeline — deterministic scripts.** Idempotent Python turns sources
   into intermediate artifacts: `mapper` (taxonomy + DNA → site
   blueprint), `generator` (blueprint → per-page specs), `tech_deploy`
   (sitemap.xml + robots.txt). Re-running yields identical output.

3. **Database — runtime copy.** A `sync` step loads yaml + artifacts
   into PostgreSQL; from then on the CMS reads only from the DB (parsing
   yaml per HTTP request is slow). The DB enables fast SSR reads, manual
   edits, change history, multi-tenancy, and analytics.

4. **Admin — SQLAdmin + RBAC.** A web UI over the ORM models: list /
   form / filters per model, custom actions (re-render, trigger
   generation), authorization with roles (owner / admin / editor /
   viewer) and per-row tenant scoping.

5. **Frontend — server-side rendering (Jinja2 + HTMX).** Every page is
   rendered on the server; the browser receives complete HTML with no
   JS content-loading. Critical for GEO: AI crawlers parse HTML without
   executing JS. A page = a DB record + an ordered list of typed blocks
   + meta (title, description, Schema.org JSON-LD). Brand surfaces use
   the client brand; geo surfaces use the branch — no client hardcoding.

6. **Reverse proxy — Caddy.** One instance routes each domain to the
   backend by Host header, terminates automatic Let's Encrypt SSL per
   domain, and handles HTTP→HTTPS, compression, and static caching.

## Data flow (onboarding → live)

```
inputs (catalog, geo) + niche DNA
        |  deterministic pipeline
        v
per-page specs + sitemap + robots
        |  sync
        v
PostgreSQL (content + per-field source)
        |--> editor (admin UI: edit / preview / publish)
        `--> deploy -> versioned build -> Caddy -> visitor / AI crawler
```

## DB schema (sketch)

All tables carry `tenant_id` + timestamps (omitted below).

- **clients** — slug, name, brand_name, niche, plan, active
- **branches** — client_id, slug, name, domain (unique), city, district,
  address, metros, og_image_url
- **services** — branch_id, category, slug, name, price, source, order
- **pages** — branch_id, slug, template_name, title, description, h1,
  canonical, meta_json, schema_org_json_ld, published, source, locked
- **blocks** — page_id, block_type, order, content_json, source
- **lsi_phrases** — page_id, phrase, volume, source
- **users** — email, role, tenant_scope, password_hash
- **audit_log** — user_id, entity, action, diff_json, at
- **deploy_history** — client_id, branch_id, version, status, log
- **jobs** — type, payload, status (async work: content generation, etc.)

## Source tracking

Every field that can come from multiple origins records a `source`
(`yaml` / `csv` / `llm` / `manual` / `api`). On `sync`: a record whose
source ≠ `yaml` is skipped; new or `yaml`-sourced records are updated —
per field, not per row. "Reset to auto" returns a field to `yaml` so the
next sync refreshes it. This is what lets regeneration and human editing
coexist safely.

## Multi-tenancy

One DB to start; every query is automatically scoped with
`WHERE tenant_id = ?` via middleware + an ORM filter event. When a
client outgrows the shared resource: spin up a separate compose
instance, ETL-copy that tenant's rows, point its domains at the new
instance through the top-level proxy, drop the old copy. The code always
filters, so escalation is ETL — not a rewrite.

## Deploy lifecycle

1. **Pre-flight validate** — required fields present, no broken links,
   sitemap valid.
2. **Build** — render all tenant pages into a versioned directory.
3. **Atomic switch** — the proxy flips to the new directory by symlink
   replace.
4. **Health check** — fetch `/`, `/sitemap.xml`, `/robots.txt` + a few
   random pages; a non-200 rolls back to the previous build.
5. **Record** — write to `deploy_history`. The last N builds are
   retained for one-click rollback.

## Reliability

- **FileLock** — a cross-platform advisory lock (sidecar lockfile with
  pid + timestamp, stale-reclaim when the owner is dead) serializes
  load-merge-write on shared semantic banks, preventing lost updates
  when parallel jobs touch the same file.
- **Atomic writes** — each write is tmpfile + `os.replace`.
- **Retry** — transient failures are retried around the lock.

## SEO endpoints

Each published domain serves: SSR home + service / category / article
pages, `/sitemap.xml`, a forward-compatible `/sitemap-index.xml`,
`/robots.txt` (with Sitemap lines), and a brand-styled 404 (HTML or
JSON). A CI smoke step exercises these contracts via in-process ASGI on
every push, failing before merge if a contract breaks.

## Testing

A test pyramid: unit (pipeline functions, models), integration (API via
in-process ASGI + SQLite, plus the same suite on real PostgreSQL for
prod parity), smoke endpoints (SEO contracts), and post-deploy health
checks. CI runs pytest + the full pipeline + SEO guards (no duplicate
H1, sitemap-equals-pages, content filters) on every push.

## What is deliberately NOT in the architecture

SPA frameworks (hurt TTFB and GEO), a headless CMS (extra stack),
microservices (overkill at this scale), a custom ORM / queue / RBAC
(SQLAlchemy / arq / SQLAdmin already fit). The system stays a lean
monolith with a clear path to scale out only where needed.
