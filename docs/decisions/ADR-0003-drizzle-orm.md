# ADR-0003 — ORM: Drizzle

- **Status:** Accepted · **Date:** 2026-08-22 · **Ledger:** D-01 (context), unanimous across all 8 source plans

## Context
TypeScript monorepo; Zod-first contracts; PostgreSQL; price history is a time series
(window functions, partial indexes); AI seasons benefit from a small, readable data layer.

## Decision
Drizzle ORM, with migrations from the first commit (`drizzle-kit`; never `db push` to prod).
NestJS integration via our own ~20-line `DrizzleModule` provider — no third-party wrapper
package (abandonment risk eliminated).

## Rationale
SQL-first (generated SQL is readable; window functions natural); ~zero runtime overhead and no
codegen engine; `drizzle-zod` completes the single type chain **DB schema → Zod → OpenAPI → FE
types** — change a column and TypeScript breaks every stale consumer; migrations are plain SQL.

## Rejected alternatives
- **Prisma:** its own schema DSL breaks the single-source Zod chain; heavy engine/cold start.
- **TypeORM:** maintenance history, weak type safety, unreliable migration generation.
- **MikroORM:** genuine enterprise patterns (UoW/Identity Map) but conceptual weight a solo
  project does not need.

## Consequences
(+) One type chain end to end. (−) Relational query API is younger; no first-party Nest module
(solved by our own provider).

## Reopen signal
If heavy DDD aggregates become real → re-evaluate MikroORM.
