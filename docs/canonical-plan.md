# AI Price Tracker — Canonical Plan

> The single authoritative plan. Every statement here is a **decided** item backed by
> [decision-ledger.md](decision-ledger.md) (D-xx / N-xx / O-xx references) and the wave-1 ADRs in
> [decisions/](decisions/). Changing anything here means updating the ledger first.
> Details live in ADRs, specs, and runbooks — not in this file. (Target size: ≤ 350 lines.)

---

## 1. Product

**Problem.** Prices, stock, and listing states of products on different Turkish sites (notebooks,
used cars, new-car price lists) cannot be tracked by hand; drops and restocks are missed.

**Solution.** A tracker that watches product/listing URLs on a schedule, detects semantic changes,
notifies automatically via **Telegram**, answers manual queries, and offers a dashboard
(web + PWA, later Tauri desktop/Android/iOS). Items are added two ways: direct URL, or free-text →
AI-resolved candidates that a **human must approve** (never auto-tracked).

**Secondary goals.** (a) Learn and practice enterprise workflows (PM, Scrumban, ADR, spec-driven
development) with Markdown artifacts only — no external tools; (b) develop via AI seasons
(planner / implementer / reviewer) with all knowledge flowing through files, never chat memory;
(c) keep a portfolio-grade process archive. Docs are **English** (D-22).

**MVP categories & pilots (D-07, O-06/O-08/O-10).** Seed targets:
[product/watchlist.md](product/watchlist.md). Sequenced strictly:
1. Notebook — **ASUS E-Store is the Phase −1 spike's preferred pilot candidate** (O-08), subject
   to the five Phase-1 entry conditions recorded in [roadmap.md](roadmap.md); Vatan remains the
   alternate candidate.
2. Used car — arabam.com single-listing URLs (**category retained; source currently
   unsupported-blocked pending a permitted/authorized first-party route**, O-14).
3. New-car OEM price lists — **five brands**: Kia TR (Sportage) + Toyota TR (RAV4) first, then
   VW TR (Tayron), Škoda TR (Kodiaq), BMW TR (X3); PDF-capable. A temporary priced Toyota Corolla
   Cross canary from the same `fiyat_v3.xml` feed is included under O-15 and is archived, never
   deleted, when RAV4 pricing returns.

Category/list-URL tracking, SKU/trim discovery, and dealer-inventory depth form one
**Phase 4 epic** (O-11, D-29), not MVP.
sahibinden.com is **Phase 4** and reopens only if all three pass: ~$10/mo proxy budget, 1 month
stable MVP, and a documented terms/access review confirming the intended method is permitted or
authorized. Blocked/protected sources remain **unsupported** — no bypassing, ever (D-06/D-07).

**i18n (D-23, owner decision).** EN/TR/AR on **all** user-facing surfaces: dashboard UI,
Telegram bot replies, notifications. Shared framework-neutral `i18next` catalogs/core;
`react-i18next` only binds React. Bot/backend render from the core without React. Arabic ⇒
RTL-safe UI from day 1; locale fallback, plural/formatting, and RTL are tested. Scraped titles
remain untranslated source data. The UI spike replaces the unverified effort estimate.

**Cross-site comparison (O-02).** In MVP: one `tracking_target` groups N `source_entries`;
the dashboard shows the same item across sites. Linking is always human-approved — automatic
product matching never happens (D-13).

---

## 2. Architecture

```
             ┌── HOME SERVER (Docker, residential IP) ────────────────────────┐
             │  apps/api    NestJS modular monolith                           │
             │    ├─ bot module (grammY, LONG-POLLING)  /add /find /check ... │
             │    └─ REST /api/v1 + OpenAPI (from Zod)                        │
             │  apps/worker in-process scheduler → run-due-checks             │
             └──────┬──────────────────────────────┬──────────────────────────┘
                    │ writes/reads                 │ primary-cycle ping
             Neon PostgreSQL (free tier)     healthchecks.io: primary check
                    ▲
                    │ same DB, same CLI + url-group fence / entry effect leases
             GitHub Actions BACKUP LANE (public repo, 60-min cron):
               after success, ping distinct backup check (including no-op)
               run-due-checks --lane=backup  → skip sources fresh on primary (< 3 h)
                                             → claim stale/due entries → scrape + notify

             Clients: browser SPA / PWA (via Tailscale + Tailscale Serve HTTPS)
                      Tauri desktop → Android APK → iOS build/simulator (CI macOS)
```

- **Runtime (D-02, ADR-0002):** home server primary (residential IP = real scraping advantage,
  $0), GitHub Actions as backup. Freshness is evaluated per adapter/source. An expiring
  `fetch_leases(adapter_key, canonical_url)` row fences each URL group across both lanes; the same
  fresh token fences every claimed member's effects. Separate healthchecks reveal either lane
  going silent.
- **Bot (D-03, ADR-0005):** grammY long-polling; polling XOR webhook — commands are down while the
  primary is down (accepted risk, N-01). 24/7 commands = future *migration* to CF Workers webhook.
- **Remote access/auth (D-09/D-10):** Tailscale Serve HTTPS is the browser perimeter; the API
  binds to loopback and its guard allowlists Serve-injected identity headers. No browser secret.
- **`run-due-checks` (D-26):** one replay-safe command (claim → scrape → compare → persist →
  enqueue notification) is called by the home scheduler, backup lane, and manual `/check`.
  Concurrent calls are expected; leases and unique effect keys make them safe. No queue yet.
- **Realtime (D-30):** a thin NestJS WebSocket "data-changed" ping tells open dashboards to
  refetch; TanStack Query polling remains permanently enabled — and is the only path during
  backup-only operation. Actual Windows + Tailscale Serve stability is an evidence gate, not an
  assumption; reconnect is bounded and falls back to polling.

## 3. Stack (locked)

| Layer | Choice | Ledger |
|---|---|---|
| Backend | NestJS modular monolith (no microservices) | D-05 |
| Validation | Zod 4 + nestjs-zod — single schema source → OpenAPI → FE types | D-14 |
| ORM / DB | Drizzle + **PostgreSQL from day 1** (Neon runtime, Docker local) | D-01, ADR-0001/0003 |
| Frontend | React 19 + Vite + TanStack Router/Query/Table + shadcn/ui (`dashboard-01`) + Tailwind | consensus |
| State | TanStack Query (server) + Zustand (client UI only) | consensus |
| i18n | shared i18next core + React binding; EN/TR/AR, RTL-safe | D-23 |
| HTTP client | Zod-first typed fetch; normal web fetch; optional host-scoped Tauri plugin transport | D-15 |
| Bot | grammY | D-03 |
| Realtime | Thin NestJS WS invalidation channel (no per-entity streams) | D-30 |
| Desktop/Mobile | Tauri 2 (Flutter rejected); PWA from the same build | D-08 |
| Monorepo | Nx + pnpm, `enforce-module-boundaries` | D-04, ADR-0004 |
| Tests | Vitest everywhere, Testcontainers (real PG), RTL+MSW, Playwright, fast-check (targeted) | D-17 |
| AI | Own router: shared core + thin Gemini/DeepSeek adapters | D-16 |

**Monorepo layout (lean — D-04):** `apps/{api,worker,web}` (+`native` at the Tauri phase),
`packages/{contracts,domain,db,scraping,ai,i18n,notifications,testing}`. Thin entrypoints over shared
services; no empty scaffold apps. `packages/domain` depends on nothing (enforced).
Runtime and administrative DB URLs are distinct: `DATABASE_DIRECT_URL` is unpooled and used for
migrations/`pg_dump`; application connections use the separately validated runtime URL (N-15).

## 4. Data model (core)

`workspaces` · `users(locale, telegram_chat_id)` ·
  `tracking_targets(workspace_id, kind, title, target_price_minor, archived_at)` ·
  `source_entries(target_id, site, url, selector?, adapter_key, health, last_checked_at,
  next_check_at, lease_owner, lease_until)` ·
  `fetch_leases(adapter_key, canonical_url, lease_owner, lease_until)` with primary key
  (`adapter_key`, `canonical_url`) ·
  `observations(entry_id, prev_observation_id?, price_minor, list_price_minor?, currency, in_stock?,
  seller_label?, snapshot_hash, extraction_method, confidence, observed_at)` ·
  `change_events(observation_id, kind, …)` ·
  `notification_deliveries(recipient_id, dedupe_key, channel, status, lease_until, attempts,
  sent_at)` ·
  `scrape_runs(entry_id?, kind, adapter_key, lane, status, started_at, duration, error_class)` —
  `kind` separates ordinary checks from `probe` runs; `entry_id` is null **only** for `probe`-kind
  rows, enforced as `CHECK (entry_id IS NOT NULL OR kind = 'probe')` (the arabam reopen probe,
  O-14) ·
  `ai_calls(provider, model, purpose, tokens, cost, latency, ok)` · `app_settings(ai_routes …)` ·
  `discovery_requests/candidates` (AI add flow).

**Invariants (full list in [CONTEXT.md](CONTEXT.md)):**
- Money = **bigint minor units (kuruş) + ISO-4217 currency**; API serializes as string; float never (D-11).
- Time = UTC `timestamptz` everywhere; localize at presentation.
- `workspace_id` on **aggregate roots only** (D-10); single seed workspace in MVP.
- Source-entry identity is (`url`, `selector`) scoped per target: URL alone is not unique within a
  target because multiple ASUS entries can share the catalogue endpoint (O-19). Null selectors must
  not be treated as distinct; use `UNIQUE NULLS NOT DISTINCT` or an equivalent partial index. The
  Phase-0 schema pins the mechanism and must not add a `UNIQUE(url)` constraint. O-19's premise is
  fetch coalescing by (`adapter_key`, canonical `url`): the worker atomically acquires that key's
  `fetch_leases` row, then writes the same fresh fencing token to every currently due member entry.
  The live group fence also blocks a second fetch for a member that becomes due or is added during
  the first fetch; entry-lock ordering alone is insufficient and is not an allowed substitute.
  One response serves all claimed selectors. The walking-skeleton spec pins this transaction and
  its two-entry/two-lane concurrency test is a Phase-1 gate item. `canonical_url` is emitted by the
  adapter and persisted as `source_entries.url` during preview/approval; the shared layer never
  guesses which query parts are disposable.
- `observations` written **only on semantic change** (`snapshot_hash`); otherwise update
  `last_checked_at`. Every attempt goes to `scrape_runs` (D-12).
- `snapshot_hash` always covers exactly (`price_minor`, `list_price_minor`, `currency`, `in_stock`)
  for every source, with `list_price_minor` null outside OEM dual-price sources and `in_stock`
  tri-state (`true`/`false`/`null`). For OEM dual-price observations, `price_minor` is the
  effective advertised price (campaign when published, otherwise list), and `list_price_minor`
  preserves the list price. Null `in_stock` means the source publishes no availability and
  participates in `snapshot_hash` as its own state. This explicit field is an input to the future
  OEM-adapter spec: it detects campaign start/end and list-price changes while target rules stay on
  the payable price. Raw `seller_label` remains display/audit text until stable Seller identity
  exists; label-only edits are not semantic events.
- Replay keys are concrete: observations form a per-entry `prev_observation_id` chain enforced by
  `UNIQUE NULLS NOT DISTINCT (entry_id, prev_observation_id)`, one canonical `change_event` per
  accepted observation, and a deterministic `dedupe_key` derived from the upserted event plus
  (`channel`, `recipient`, later rule id) (D-05/D-26).
- Due work is claimed with a renewable/expiring url-group fence plus entry effect leases;
  `lease_until` covers the full protected operation (including backup upload/finalize), or is
  renewed throughout it. One fresh UUID is written to the claimed
  `fetch_leases` row and every due group member, and the effect transaction re-validates both token
  scopes plus expiry, aborting as a no-op if either changed; all checks use DB `now()` (D-02/D-26).
- Change detection and swing guards compare against the last **accepted** observation; suspicious or
  AI-quarantined values never become the baseline and never alert unconfirmed (D-06/D-26).
- Notifications: at-least-once outbox (`pending → sending lease → sent/failed`) + unique
  `dedupe_key`. A narrow post-send/pre-commit duplicate window is accepted for MVP (D-05).
- Failure diagnostics: sanitized + size-capped; local `diagnostics/failures/` files or diagnostic
  Actions artifacts with explicit 7-day retention; unsanitized full HTML is forbidden and
  diagnostics are **never in Neon** (D-12). The initial accepted baseline, every accepted change,
  and every suspicious/held candidate get a sanitized, size-capped local snapshot. Primary first
  atomically installs it as `pending`, commits effects only after that succeeds, then reconciles its
  state before a healthy ping. All snapshots survive 35 days; rotation then keeps the two newest
  `accepted` baseline snapshots and every unresolved `held`/`pending` snapshot, so held attempts
  cannot evict the before/after chain. The extracted region is never truncated. Each accepted/held
  backup candidate contributes one envelope to its workflow execution's single 30-day artifact.
  Batch `prepare → upload → finalize` has no semantic effects before upload; afterward finalize
  independently revalidates both lease scopes for each candidate. A zero-envelope no-op skips upload.
  Failures log and retry affected work from a fresh fetch, so no backup alert lacks evidence.
  Before reporting a healthy cycle, the primary's authenticated/idempotent ingester verifies the
  exact workflow, API digest, archive shape, and envelope identifiers/hash, resolves the terminal
  run states, then atomically places every unseen envelope in the local store. Failures retry and
  suppress that health ping. This preserves both lanes' chain when primary returns within 30 days;
  a longer outage is accepted as capable of losing already-expired evidence (D-12).
- PostgreSQL `CHECK` constraints (price > 0, valid enums) back Zod + sanity guards as the final
  defense layer; `change_events.kind` is extensible — NEW_SKU / NEW_TRIM / NEW_SELLER values
  arrive with their Phase-4 features (D-29).
- `seller_label` is nullable point-in-time display/audit data, not identity. N simultaneous
  marketplace sellers require the deferred Seller + source-offer model (D-29).

## 5. Scraping (the riskiest part — D-06, ADR-0006)

Ladder, cheapest first: **0)** official API/feed → **1)** native fetch → **2)** JSON-LD / meta /
embedded state (`__NEXT_DATA__`) → **3)** site adapter + Cheerio → **4)** PDF (unpdf, OEM lists)
→ **5)** Playwright only when forced. Crawlee deferred until list crawling is real.

- `SourceAdapter` port: `canHandle(url)` / `discover?(query)` / `fetch(ctx)` / **pure** `parse(raw)`
  (fixture-testable, network-free) + per-domain policy (rate limit, jitter, robots).
- Hygiene (N-09): robots.txt, per-domain rate limit + randomized intervals, realistic UA,
  ETag/If-Modified-Since, consecutive-failure counter → auto-pause + operator Telegram alert.
  Counters are keyed by **source entry + lane + error class**; adapter `health_score` is aggregated.
  A backup-lane transport/block failure cannot pause primary, and one bad URL cannot pause every
  entry for an adapter. Jitter is meaningful (order of ±10–20%), never a token few seconds.
- Intervals (O-13): 60-min default + jitter; per-source bases (e.g. OEM daily) complement
  per-target overrides. A proven cheap rung 0–2 path may use a 10–15 min minimum only when
  source terms/published limits and adapter rate policy allow; Cheerio/PDF/browser stays ≥60 min.
- **Sanity guards:** price ≤ 0 or >70% swing on either price field, or a campaign-presence flip
  (D-12) → flag suspicious, hold the alert, ask the human. Wrong price is worse than silence.
- **AI extraction is quarantined:** last-resort AI-extracted values carry
  `extraction_method='ai'` + confidence, never fire alerts without human confirmation; main job is
  drafting parser repairs (human-approved).
- **Fixture tests catch code regressions only — they cannot see live-site changes (N-03).**
  Live drift is caught by the nightly live-smoke job (1 real URL per adapter, non-CI-blocking,
  failure → Telegram).

## 6. AI layer (D-16)

- **Own router:** shared provider core + thin `GeminiAdapter`/`DeepSeekAdapter`. Compatible
  transport is reused, while structured outputs, parameters, errors, rate limits, and
  capabilities are normalized. A third provider requires a capability/contract test.
- **Task-based routes over a default chain:** normalize = cheap; candidate ranking = cheap,
  escalate on low confidence; parser repair = strong + human approval. Default chain
  `gemini flash-lite → gemini flash → deepseek v4 flash`; order lives in DB, editable via
  `/model` and dashboard; model IDs are config, re-verified at integration (N-08).
- Retries with schema feedback (all outputs Zod-validated), then provider fallback;
  circuit breaker on consecutive failures.
- Every call is logged to `ai_calls`. Mandatory limits: calls/run/day, input/output tokens,
  retries, and provider concurrency. **O-03 is reopened:** before live calls, document the exact
  API credential, included/free quota, and pay-per-use terms; the owner then sets the monetary
  cap policy. Privacy cleared (O-04): public pages only, no personal data in prompts.
- Build a small **eval set (20–50 real samples)** before locking the chain order.
- Product adding: **Path A** direct URL → adapter → preview → approve. **Path B** free text → AI
  normalize → multi-source search → ranked candidates (Telegram inline keyboard / dashboard
  modal) → user picks → Path A preview. **AI proposes, human approves — always.** The preview shows
  the raw source string beside the parsed minor-unit value, and when the workspace already holds a
  source entry with the same (`url`, `selector`) it says so and requires explicit confirmation
  before a second tracking target is created for it — duplicate targets fire duplicate alerts that
  the per-target cooldown cannot suppress.

## 7. API & clients

- Single **`/api/v1`** (NestJS URI versioning) for every client; no BFF (D-14).
- Zod is the single contract source; OpenAPI derived from it; **OpenAPI snapshot test in CI**
  breaks on contract drift (D-14, D-27).
- Fetch wrapper: timeout/AbortSignal, error taxonomy, Zod response validation. Browser/PWA uses
  normal fetch with an exact CORS allowlist. Tauri uses normal fetch unless a verified native
  constraint requires `plugin-http`; then its capability is scoped to exact HTTPS hosts (D-15).
- **PWA:** same web build + manifest; service worker only in the web profile (static app-shell
  cache only, never API responses); reached over Tailscale Serve HTTPS (D-08, D-09).
- **Realtime:** one WS invalidation ping → clients refetch. Upgrade identity is allowlisted; no
  query-string secret. Bounded reconnect + polling fallback are mandatory (D-30).
- Auth (MVP, D-10): Telegram `chat_id` allowlist + Tailscale identity allowlist behind an
  `AuthGuard`; API listens on loopback behind Serve. No static token in browser code/storage.
  Optional static token is CLI/server-to-server only. Swap the guard later, controllers unchanged.

## 8. Testing (D-17)

| Layer | Tooling | Notes |
|---|---|---|
| Unit + property | Vitest + fast-check | property tests only where invariants are dense: price parsing, threshold math, dedupe/idempotency |
| Adapter contract | fixtures per site | regression only (N-03); paired with nightly live smoke |
| Backend integration | Testcontainers **real Postgres** + supertest | in-memory SQLite hides the SQL bugs we want |
| Frontend integration | RTL + TanStack Query + MSW | page-level: fetch → render → interact → invalidate |
| Realtime resilience | actual Windows + Tailscale Serve | 30-min soak, identity rejection, network-loss reconnect, polling takeover |
| E2E | Playwright (3–5 critical flows) | add item → list → history; RTL smoke included |
| Native smoke | tauri-driver | only at the native phase |

Coverage gates **only** on parser/domain packages — no repo-wide % gate.

## 9. Process (D-18 … D-22)

- **Scrumban:** 1-week cycles, **WIP = 1**, sprint goal + demo/review + short retro; no daily
  standup. Story points kept lightweight **+ actual cycle time recorded**; calibration after ~3
  cycles. **No-Fiction rule:** every artifact is generated from real data. One process policy
  ([records/process-policy.md](records/process-policy.md)) records intentionally omitted
  ceremonies; do not create one document per non-event.
- **Specs (D-21):** Kiro IDE generates `requirements.md` (EARS) / `design.md` / `tasks.md` —
  **only for work above a few hours or spanning layers**; small work = board entry + commit
  trail. Season outputs: `handoff.md` + `review.md` alongside the spec. Prompt kit:
  [prompts/](prompts/). Test plan is a section of `design.md`.
- **Reviews (D-19):** risk-based during the phase, plus two fresh reviews at its gate. High-risk
  stories get an immediate independent/adversarial review before dependent work; medium-risk
  work gets one independent story/epic review; low-risk work uses automation + preflight.
  In-phase reviews use exact commit ranges. The phase gate reviews `main..dev`: informed review
  plus independent changes review, followed by one reconciliation record.
- **Git (D-20):** `main` + `dev` only; no PRs, no feature branches. Conventional commits +
  `PT-###`; every commit working; mid-phase tags as restore points. CI: `dev` push → lint/
  typecheck/unit/integration; `main` merge → e2e/build/release. `main..dev` = the phase-review diff.
- **Tracking (O-07 resolved — deferred automation):** `work/board.md` is the single manual
  status source through the spike and walking skeleton; status is NOT duplicated in front-matter.
  At ≥5 active stories or the first real status drift, status moves into story front-matter and
  a ~60-line `board.ts` + lefthook + `pm-check` generate and guard the board.
- **Docs:** `AGENTS.md` = canonical repo contract; `CLAUDE.md` = tool-specific deltas only.
  `CONTEXT.md` = invariants, **hard cap 200 lines**, read by every AI season first.
  ADRs: 7 core now; everything else promoted from the ledger on touch. Every ADR carries a
  **reopen signal** (N-10).

## 10. Roadmap

Phase-based and **date-free** (O-05). Full detail: [roadmap.md](roadmap.md).

| Phase | Goal (one line) |
|---|---|
| **-1 Spike** | 1–2 days, timeboxed: probe Vatan, ASUS, arabam, and all 5 OEM endpoints using watchlist targets where carried; use an off-watchlist Vatan control — before foundation |
| **0 Foundation** | Nx workspace, Docker PG, Drizzle schema v1, two-tier CI, docs seeds, 7 ADRs |
| **1 Walking skeleton** | one URL → extract → PG → change → Telegram; concurrent-lane lease + both healthchecks live |
| **2 MVP build-out** | 5 OEM adapters (arabam blocked, O-14), rules+dedupe, dashboard (+comparison, WS ping), AI discovery, i18n EN/TR/AR, PWA over Tailscale |
| **3 Native** | Tauri desktop → Android signed APK → iOS build/simulator (CI macOS, public repo) |
| **4 Post-MVP / hardening** | category/list tracking + SKU/trim discovery + dealer inventory + analytics; sahibinden spike only after all budget/stability/access-review gates; Crawlee if needed |
| **5 SaaS-readiness** | real auth/pairing, quotas, Astro landing, billing, KVKK/GDPR |
| **6 Enterprise** | SSO/SCIM, RBAC, audit, SLA/SLO, DR — only on real demand |

**Iron sequencing rule (D-07):** nothing broadens until the walking skeleton's first real
notification works end to end. "No screens, no beauty — a price lands in the DB and a message
arrives."

## 11. Not now (YAGNI — unanimous unless noted)

Microservices · gRPC/GraphQL · event bus/CQRS/event sourcing/saga (D-05) · Redis/BullMQ (trigger:
multi-worker or >200 targets) · membership/RBAC/billing · k8s/Terraform · feature-flag service ·
design-system package (until a 2nd app's UI diverges) · generic JobQueue framework (D-26) ·
proxy/stealth stack (sahibinden only after every D-07 gate) · store distribution / paid Apple account ·
separate BFFs · OTel backend (structured logs suffice) · full per-entity WS streaming (thin
invalidation channel only, D-30).

## 12. Verification ritual

Every phase has an exit gate file (`docs/roadmap.md` gates): each criterion answered with
**evidence** — command output, screenshot path, DB query result, commit SHA. No evidence, no gate.
Continuous gates: two-tier CI green · OpenAPI snapshot unchanged · board current (generated +
`pm-check` once the O-07 trigger fires) · nightly live smoke green (or Telegram alert fired).
