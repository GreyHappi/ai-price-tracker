# ADR-0007 — Money and time representation

- **Status:** Accepted · **Date:** 2026-08-22 · **Ledger:** D-11

## Context
Every core feature compares prices (diffs, thresholds, percentages). Drizzle returns Postgres
`NUMERIC` as strings, a classic silent-bug source. Timestamps flow between scraper, DB, API, and
three UI locales.

## Decision
- Money = **`bigint` minor units (kuruş)** + ISO-4217 `currency` column. Never float, never
  NUMERIC. The API serializes money as a string; a `Money` value object handles arithmetic and
  formatting in `packages/domain`.
- Time = **UTC `timestamptz`** everywhere; localization only at the presentation layer.

## Rationale
Integer arithmetic makes diff/threshold math deterministic and eliminates the float error class
entirely; it also sidesteps Drizzle's numeric-string trap. UTC-everywhere avoids timezone drift
across lanes and clients.

## Rejected alternatives
- `NUMERIC(12,2)` + decimal strings (linear-lark, plan-2): valid, but string math in app code is
  less deterministic and error-prone.
- Float: never an option.

## Consequences
(+) Trivial, exact comparisons. (−) Conversion happens at the edges (parser input, UI display) —
owned by the `Money` value object.

## Reopen signal
Multi-currency aggregation needs (SaaS phase) → add FX tables; representation stays minor-unit.
