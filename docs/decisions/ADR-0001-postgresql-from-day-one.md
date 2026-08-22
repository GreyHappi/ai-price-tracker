# ADR-0001 — PostgreSQL from day 1

- **Status:** Accepted · **Date:** 2026-08-22 · **Ledger:** D-01

## Context
Greenfield project; scraper, API, and bot write to the DB concurrently; a SaaS path exists;
Neon's free Postgres tier is available; Drizzle is the ORM (ADR-0003).

## Decision
PostgreSQL from the first migration. Neon (runtime) + Docker Postgres (local dev & integration
tests). No SQLite/Turso stage, no dual-dialect configuration.

## Rationale
1. Drizzle schemas are dialect-specific (`sqliteTable` ≠ `pgTable`) — "migrate later" means
   rewriting schemas, migrations, and type tests. It is not a driver swap.
2. SQLite's single-writer lock (`SQLITE_BUSY`) is exactly this workload's weak spot
   (3 concurrent writers).
3. Type loss in SQLite: `timestamptz`, `jsonb`, partial indexes, window-function-friendly
   time series.
4. A cloud-reachable DB is what makes the dual-lane runtime (ADR-0002) possible at all.

## Rejected alternatives
- **SQLite / Turso start** (recursive-lamport, plan-1): migration cost understated.
- **Dual dev-SQLite / prod-PG** (replicated-crayon): two schema sources to keep in sync forever.

## Consequences
(+) One dialect forever; both lanes reach the same DB. (−) Docker required locally; Neon free
caps (0.5 GB / 100 CU-h) — mitigated by change-only writes and the no-raw-HTML-in-DB rule (D-12).

## Reopen signal
Neon free limits exceeded → self-hosted Postgres on the home server or a paid tier.
