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
- Source-entry identity is (`url`, `selector`) scoped per target — `url` alone is never unique
  (multiple ASUS entries share the catalogue endpoint, O-19). Null selectors are not distinct:
  `UNIQUE NULLS NOT DISTINCT` or an equivalent partial index; the Phase-0 schema pins it.
- `observations` are written **only on semantic change**: `snapshot_hash` covers exactly
  (`price_minor`, `list_price_minor`, `currency`, `in_stock`); `in_stock` is tri-state (null =
  not published). For OEM dual-price sources, `price_minor` is the effective advertised price
  (campaign when published, otherwise list) and nullable `list_price_minor` preserves the list
  price. Raw `seller_label` is display/audit snapshot text, not canonical identity, so a
  label-only diff creates neither an observation nor a `change_event` in MVP; stable seller
  identity joins the hash only with the Phase-4 Seller model. Otherwise update `last_checked_at`.
  Every attempt is logged to `scrape_runs` (with `lane`).
- Replay keys are concrete: each observation stores `prev_observation_id`; one chain per entry is
  enforced by `UNIQUE NULLS NOT DISTINCT (entry_id, prev_observation_id)` on PG15+ (or equivalent
  partial indexes), giving one head per entry and one successor per predecessor. One canonical
  `change_event` exists per accepted observation (unique `observation_id`). It is upserted/retrieved
  before `dedupe_key` is derived from (`change_event_id`, `channel`, `recipient`, + rule id once
  Phase-2 rules exist) — never timestamps, attempt counters, or ids minted anew by a retry.
- Due work is fenced first by an expiring `fetch_leases` row keyed by (`adapter_key`, canonical
  `url`), then by the affected `source_entries`: one claim transaction acquires that url-group
  fence and writes the same fresh UUID to every currently due member's `lease_owner`. The token is
  never a stable worker/lane id. Group and entry `lease_until` values cover the whole protected
  operation — including backup upload/finalize — or are renewed throughout it; every effect
  transaction matches both unexpired tokens, aborting as a no-op if either check fails. Every
  freshness/due/lease comparison uses database time (SQL `now()`), never a process or runner clock.
- Notifications are at-least-once outbox-lite: unique `pending` → leased `sending` →
  `sent`/`failed`, with attempt count. A rare post-send/pre-commit duplicate is accepted in MVP.
  Alert cooldown is scoped per (`tracking_target`, rule, channel, recipient) with a configurable
  minimum gap (default set by the Phase-2 alert-rules spec); it suppresses the same rule re-firing
  across distinct `change_events` inside the window.
- Failure diagnostics are sanitized and size-capped. Local `diagnostics/failures/` files rotate
  after 7 days; diagnostic Actions artifacts set `retention-days: 7`. Unsanitized full HTML is
  forbidden; diagnostics never enter Neon. The initial accepted baseline, every accepted change,
  and every suspicious/held candidate get a sanitized, size-capped page snapshot in primary's
  separate `snapshots/` directory (never Neon). Primary atomically installs a `pending` snapshot
  before semantic effects, then reconciles its state from the committed run before the cycle is
  healthy. All snapshots survive 35 days; rotation thereafter always keeps the two newest
  `accepted` baseline snapshots plus every unresolved `held` or `pending` snapshot, so held attempts
  cannot evict the current/predecessor chain. The extracted field or its containing region is
  preserved verbatim and never truncated by the size cap.
  Each backup candidate accepted or held by one workflow execution contributes one `snapshot-v1`
  envelope to that execution's single `snapshots-<github-run-id>-<run-attempt>` Actions artifact
  (`retention-days: 30`). Backup processing is batch `prepare → upload → finalize`: prepare writes
  all envelopes/continuations but no baseline, observation/event/delivery; after one upload,
  finalize independently revalidates each live group/entry fence before its effects. A zero-envelope
  no-op skips upload. Upload/finalize failure logs, produces no unfenced alert/baseline move, and
  retries affected work from a fresh fetch; an alert can never outrun its artifact.
  Before a primary cycle is healthy, its authenticated, idempotent ingester enumerates every
  non-expired artifact from the exact backup workflow, verifies the API digest and every envelope's
  identifiers/hash, rejects unexpected archive paths/shapes, resolves terminal run states, and
  atomically places each validated envelope in the local
  `snapshots/` store; failure retries and withholds the primary health ping. Thus one store holds
  both lanes' chain when primary returns inside 30 days; a longer outage can still lose expired
  evidence and is an accepted D-12 risk. On-disk layout is one directory per source entry holding
  one self-contained envelope per snapshot at
  `snapshots/<entry-id>/<yyyyMMddTHHmmss.fffZ>-<scrape-run-id>-<state>.json`; state is
  `pending`, `accepted`, `held`, or `discardable`, and application transitions are atomic. Rotation
  orders/ages by the validated UTC capture time, never download/`LastWriteTime`, and does not
  manage files outside the shape (`runbooks/windows-server.md` pins the ingest and sweep).
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
- Entries sharing (`adapter_key`, canonical `url`) are **coalesced**. The claim unit is that url
  group: an atomically acquired `fetch_leases` row is the single cross-lane fence, and the winner
  claims all currently due member entries with the same fencing token. Another member becoming due
  or being added during the fetch cannot open a second fetch while the group fence is live. One
  response serves every claimed selector; the group is never split across lanes. This is O-19's
  premise, not an optimization; entry-lock ordering alone is not an allowed implementation. The
  adapter produces and persists the canonical fetch URL as `source_entries.url` during
  preview/approval; no global heuristic strips or reorders query parameters whose semantics it
  cannot know.
- Auto-pause counters are keyed by (`source_entry`, lane, error class); adapter health is an
  aggregate. A backup-only transport/block failure cannot pause primary, and one bad URL cannot by
  itself pause every entry for an adapter.
- Intervals: 60-min default + jitter; per-source bases (e.g. OEM daily) alongside per-target
  overrides. A proven cheap rung 0–2 path may use a 10–15 min minimum only when source
  terms/published limits + adapter policy allow; Cheerio/PDF/browser paths stay ≥60 min.
- `parse()` is **pure**: recorded input → Observation. Network-free, fixture-tested.
- A source's numeric format (decimal/thousands separators) is a **declared per-adapter property**,
  never a shared heuristic — the spike observed mutually ambiguous shapes (`161329`, `2.249.000`,
  `244999.000`, `3.776.000,00`). The `/add` preview shows the raw source string beside the parsed
  minor-unit value so a human can catch a misread format before the entry is approved.
- Sanity guards and change detection always compare against the last **accepted** observation:
  price ≤ 0 or a strictly >70% swing (integer math) on **either** `price_minor` or
  `list_price_minor` ⇒ mark suspicious, hold the alert, ask the human; a suspicious or
  AI-quarantined value never becomes the baseline and never fires an alert until a human accepts it
  (the walking-skeleton spec pins storage/rejection mechanics). The `in_stock` nullability is
  pinned per source; a null ↔ non-null transition is suspicious, held, and never alerted. On
  dual-price sources a null ↔ non-null transition of `list_price_minor` — a campaign appearing
  or disappearing — is suspicious the same way: held, never alerted until a human accepts it.
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
