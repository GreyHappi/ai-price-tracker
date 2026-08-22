# ADR-0004 — Monorepo: Nx + pnpm

- **Status:** Accepted · **Date:** 2026-08-22 · **Ledger:** D-04

## Context
One repo hosts api/worker/web (+ native later) and 7 shared packages. Owner's explicit learning
goal: enterprise patterns. Architecture rot is the main long-term risk of a solo AI-driven repo.

## Decision
Nx with pnpm. `@nx/enforce-module-boundaries` lint rules from day 1:
`apps/*` → `packages/*` allowed; `packages/domain` depends on **nothing**; `packages/db` only on
`contracts`. Lean layout — thin entrypoints over shared services; no empty scaffold apps
(`landing`, `pm`, `mobile` are created only when their phase starts).

## Rationale
7 of 8 source plans chose Nx. Boundary rules are the only *automatic* mechanism protecting the
architecture — documentation alone rots. `nx affected` keeps CI fast and within free minutes.

## Rejected alternatives
- **Turborepo** (federated-grove): lighter, but no boundary enforcement/generators and weaker
  enterprise signal.
- **npm / yarn:** pnpm chosen (owner decision; also every source plan's assumption).

## Consequences
(+) Mechanical architecture protection; generators available later. (−) Nx config learning curve.

## Reopen signal
None foreseen.
