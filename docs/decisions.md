[Русский](./decisions.ru.md) · **English**

# Key decisions (ADR-style)

Condensed records of the decisions that shaped the system — context and
consequences only, no client data or code.

## ADR-1: Git as the source of truth
**Context.** The system is defined by many declarative inputs (niche
taxonomies, client catalogs, geo configs, templates).
**Decision.** Keep them all as files in git; the database is a derived
runtime copy.
**Consequences.** Free audit trail (git log), trivial rollback, routine
edits without a DB, "clone = provision". Costs a `sync` step to load the
DB before serving.

## ADR-2: Deterministic pipeline, LLM only at the edges
**Context.** Site structure must be reproducible and reviewable; LLM
output is neither.
**Decision.** The pipeline (taxonomy → blueprint → page specs) is pure
and deterministic; the LLM is confined to content drafting behind an
adapter.
**Consequences.** Re-runs are diff-able and testable; provider swaps are
local; most of the system has no nondeterminism to debug.

## ADR-3: Per-field source tracking
**Context.** Regeneration must not destroy manual edits, but auto-fields
must still refresh.
**Decision.** Every multi-origin field stores a `source`
(`yaml`/`csv`/`llm`/`manual`/`api`); `sync` skips non-`yaml` fields and
updates the rest — per field, not per row. "Reset to auto" re-opens a
field.
**Consequences.** Humans and the pipeline edit the same rows safely;
this is the feature that makes large-scale generation operable.

## ADR-4: Server-side rendering over SPA
**Context.** Search is shifting to AI answer engines that read HTML
without executing JavaScript.
**Decision.** Render every page on the server (Jinja2 + HTMX), ship
complete HTML with Schema.org JSON-LD; no client-side content loading.
**Consequences.** Strong classic-SEO and GEO posture, low TTFB, one
stack. Interactivity comes from server-rendered partials, not a SPA.

## ADR-5: PostgreSQL runtime copy, separate from the git files
**Context.** Parsing dozens of yaml files per HTTP request is too slow,
and editors should not edit yaml.
**Decision.** A `sync` loads files into PostgreSQL; the app reads only
the DB at request time.
**Consequences.** Fast reads, manual editing surface, change history,
multi-tenancy, and analytics — at the cost of a sync step and a
files↔DB contract.

## ADR-6: Single-DB multi-tenancy first
**Context.** Many clients/branches, but most are small.
**Decision.** One database with `tenant_id` scoping enforced by
middleware + an ORM filter event; isolate a tenant later by ETL into a
dedicated instance.
**Consequences.** Cheap to start; the code always filters, so escalation
is data movement, not a rewrite.

## ADR-7: SQLAdmin + RBAC instead of a custom admin
**Context.** Non-technical editors need CRUD, preview, and publish.
**Decision.** Use SQLAdmin over the ORM models with role + per-row
tenant scoping, rather than building an admin from scratch.
**Consequences.** Most of the back office for free; custom actions
(re-render, generate) added as needed; less code to own.

## ADR-8: Atomic deploy with health-checked rollback
**Context.** A bad publish must never leave a live site broken.
**Decision.** Build to a versioned directory, switch by symlink replace,
health-check key URLs, roll back automatically on a non-200; keep the
last N builds.
**Consequences.** Safe, reversible publishes; one-click rollback from
history.

## ADR-9: Own FileLock for shared mutable files
**Context.** Parallel jobs for clients in the same niche read-merge-write
the same shared semantic bank, losing each other's updates.
**Decision.** A cross-platform advisory FileLock (sidecar lockfile, pid +
timestamp, stale-reclaim) serializes load-merge-write; atomic writes +
retry complete the picture.
**Consequences.** No lost updates on shared files; a clear timeout error
instead of silent corruption.

## ADR-10: Decoupled semantic source
**Context.** Semantic harvesting (demand collection, dedup, taxonomies)
is a distinct concern that wants to grow independently (multi-niche,
vector core).
**Decision.** Extract it to a separate platform; this generator consumes
a ready core through a thin `SemanticSource` seam (cache-by-default,
HTTP when upstream is available, cache fallback).
**Consequences.** The generator no longer collects semantics — it
compiles from a finished core; the two systems evolve independently.

## ADR-11: Caddy as the reverse proxy
**Context.** Dozens of domains need TLS and Host-based routing with
minimal ops.
**Decision.** A single Caddy instance routes by Host, issues automatic
Let's Encrypt certificates per domain, and handles redirects/compression.
**Consequences.** Per-domain HTTPS with near-zero config; the proxy is
also the switch point for tenant isolation later.

## ADR-12: SQLite for dev/test, PostgreSQL parity in CI
**Context.** Tests should be fast locally, but prod runs on PostgreSQL.
**Decision.** Run the suite on in-process SQLite for speed and the same
suite on a real PostgreSQL service in CI to catch DDL/behaviour
divergence.
**Consequences.** Fast local iteration without losing prod fidelity.
