# CONTEXT.md — Invariants

> **HARD CAP: 200 lines.** Every AI season reads this file first. If a change conflicts with
> anything here, stop and flag it (name the ledger D-xx). If this file needs to grow past the cap,
> split details into `docs/architecture/` and trim.

## Data

- Money = **bigint minor units (kuruş)** + ISO-4217 `currency` column. Float never. API serializes
  money as string. MVP is single-currency per `source_entry`; a parsed currency differing from the
  entry's ⇒ suspicious, hold the alert, ask the human — prices are never compared across currencies.
- Time = **UTC `timestamptz`** everywhere; localize only at presentation.
- `workspace_id` on aggregate roots only; single seed workspace in MVP.
- `observations` are written **only on semantic change**: `snapshot_hash` covers exactly
  (`price_minor`, `currency`, `in_stock`). Raw `seller_label` is display/audit snapshot text, not
  canonical identity, so a label-only diff creates neither an observation nor a `change_event` in
  MVP; stable seller identity joins the hash only with the Phase-4 Seller model. Otherwise update
  `last_checked_at`. Every attempt is logged to `scrape_runs` (with `lane`).
- Replay keys are concrete: each observation stores `prev_observation_id`; one chain per entry is
  enforced by `UNIQUE NULLS NOT DISTINCT (entry_id, prev_observation_id)` on PG15+ (or equivalent
  partial indexes), giving one head per entry and one successor per predecessor. One canonical
  `change_event` exists per accepted observation (unique `observation_id`). It is upserted/retrieved
  before `dedupe_key` is derived from (`change_event_id`, `channel`, `recipient`, + rule id once
  Phase-2 rules exist) — never timestamps, attempt counters, or ids minted anew by a retry.
- Due source entries are claimed through an expiring PostgreSQL lease (`FOR UPDATE SKIP LOCKED`);
  `lease_owner` is a fresh UUID minted **per claim**, never a stable worker/lane id, and acts as the
  fencing token. `lease_until` exceeds the adapter's maximum scrape timeout with margin (or is
  renewed mid-scrape); the effect transaction matches that exact token and an unexpired lease,
  aborting as a no-op if either check fails. Every freshness/due/lease comparison uses database
  time (SQL `now()`), never a process or runner clock.
- Notifications are at-least-once outbox-lite: unique `pending` → leased `sending` →
  `sent`/`failed`, with attempt count. A rare post-send/pre-commit duplicate is accepted in MVP.
  Alert cooldown is scoped per (`tracking_target`, rule, channel, recipient) with a configurable
  minimum gap (default set by the Phase-2 alert-rules spec); it suppresses the same rule re-firing
  across distinct `change_events` inside the window.
- Failure diagnostics are sanitized and size-capped. Local files rotate after 7 days; Actions
  artifacts set `retention-days: 7`. Unsanitized full HTML is forbidden; diagnostics never enter Neon.
- Soft-archive with `archived_at`; never hard-delete tracking history.
- PostgreSQL CHECK constraints (e.g. price > 0) back Zod and sanity guards; keep them in sync
  with schema changes.
- Runtime DB configuration and administrative access are separate: `DATABASE_DIRECT_URL` is
  unpooled and required for migrations/`pg_dump`; run neither through a pooler (drizzle-kit under
  PgBouncer transaction mode is unsafe). Startup/CI assert the migration connection is non-pooled.
- `seller_label` on an observation is nullable snapshot text, never canonical identity. Multiple
  simultaneous sellers require the deferred Seller + source-offer model.

## Scraping

- Ladder order is mandatory: official API → fetch → JSON-LD/embedded state → Cheerio adapter →
  PDF (unpdf) → Playwright **last**.
- **Never bypass CAPTCHA/login/anti-bot.** Mark such sources `unsupported`.
- Per-domain rate limit + jitter; robots.txt respected; realistic UA; ETag/If-Modified-Since.
- Auto-pause counters are keyed by (`source_entry`, lane, error class); adapter health is an
  aggregate. A backup-only transport/block failure cannot pause primary, and one bad URL cannot by
  itself pause every entry for an adapter.
- Intervals: 60-min default + jitter; per-source bases (e.g. OEM daily) alongside per-target
  overrides. A proven cheap rung 0–2 path may use a 10–15 min minimum only when source
  terms/published limits + adapter policy allow; Cheerio/PDF/browser paths stay ≥60 min.
- `parse()` is **pure**: recorded input → Observation. Network-free, fixture-tested.
- Sanity guards and change detection always compare against the last **accepted** observation:
  price ≤ 0 or a strictly >70% swing (integer math) ⇒ mark suspicious, hold the alert, ask the
  human; a suspicious or AI-quarantined value never becomes the baseline and never fires an alert
  until a human accepts it (the walking-skeleton spec pins storage/rejection mechanics).
- AI-extracted values are quarantined (`extraction_method='ai'`): never alert unconfirmed.
- Scraped text (titles, `seller_label`) is untrusted output: escaped for its context —
  parse_mode-safe (or no `parse_mode`) in Telegram, never `innerHTML`/`dangerouslySetInnerHTML`.
- Fixture tests catch **code regressions only** — live-site drift is caught by the nightly
  live-smoke job, not by fixtures.

## AI

- **AI proposes, human approves** — no auto-tracking, no auto-applied parser repairs.
- All AI outputs are Zod-validated; retry with schema feedback, then fall back to the next
  provider; circuit breaker on consecutive failures.
- Model IDs live in DB config, never in code. Every call logged to `ai_calls`
  (purpose, tokens, latency, cost).
- Provider adapters normalize capability differences; compatibility is not assumed from protocol
  shape alone. Calls/run/day, tokens, retries, and concurrency are bounded.
- No live AI calls until the exact API credential, quota, and billing terms are documented (O-03).
- No personal data in prompts; public product pages only.

## API & contracts

- Single **`/api/v1`**. Zod schemas in `packages/contracts` are the single source of truth;
  OpenAPI is derived from them and snapshot-tested in CI.
- Browser/PWA auth is the Tailscale Serve identity allowlist; API binds to loopback behind Serve.
  Never bundle/store a static API token in browser code. Static tokens are CLI/server-only.
- All HTTP goes through the shared fetch wrapper. Web uses normal fetch + exact CORS allowlist;
  native plugin transport is optional and, if enabled, scoped to exact approved HTTPS hosts.
- Bot handlers never scrape inline (grammY processes updates sequentially — one long scrape stalls
  every command): `/check` only marks entries due (`next_check_at = now()`) and replies at once;
  the scheduler/lease path does the work.
- Realtime is an optional invalidation hint, never the consistency path: validate Tailscale
  identity on WS upgrade, reconnect with bounds, and keep polling active as fallback (D-30).

## i18n

- **EN/TR/AR on ALL user-facing text**: UI, bot replies, notifications. Shared framework-neutral
  `i18next` catalogs/core; `react-i18next` is React-only. Backend renderers do not depend on React.
- UI must be **RTL-safe**: logical properties (`ms-/me-/ps-/pe-`, start/end), dir-aware
  components. Every UI story's DoD includes an RTL check.
- Test locale fallback, plurals, number/date/money formatting, and RTL. Scraped titles remain
  untranslated source data.
- Never hardcode a user-facing string.

## Process & git

- Work on `dev`; merge to `main` only at phase gates. Conventional Commits + `PT-###`.
  Every commit leaves the repo working.
- Specs only for work above a few hours or spanning layers. Deviations from a spec go to
  `handoff.md` — never silently into code.
- **No-Fiction:** artifacts are generated from real data. One process policy records ceremonies
  intentionally omitted; do not create repetitive documents for each non-event.
- Reviews are risk-based: high risk immediately, medium at story/epic end, low via automation;
  every phase gate still receives informed + independent fresh-session reviews.
- Secrets are never committed.
