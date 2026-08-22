# Roadmap

> Phase-based and **date-free** (owner decision O-05). Each phase ends at an **evidence gate**:
> every criterion is answered with proof — command output, screenshot path, DB query result, or
> commit SHA. No evidence, no gate. **Iron rule (D-07):** nothing broadens until Phase 1's first
> real notification works end to end.

## Phase −1 — Risk spike (1–2 days, throwaway code)

Prove the pilots are extractable **before** building any foundation:

- Vatan product page: price/stock via plain fetch + JSON-LD/embedded state?
- ASUS E-Store ROG SKU pages: extractable? (Notebook pilot = spike winner, Vatan vs ASUS — O-08.)
- arabam.com listing page: price/status extractable? Which ladder rung?
- Every committed OEM endpoint: Kia Sportage, Toyota RAV4, VW Tayron, Škoda Kodiaq, BMW X3.
  Record each endpoint's actual format/rung and one sample; do not assume equivalent source classes.
- All probes run against the real targets in [product/watchlist.md](product/watchlist.md) (O-09).

**Gate:** `docs/records/spike-results.md` has one evidence row for all 8 probed endpoints
(2 notebook candidates + arabam + 5 OEM): source → URL → format/rung → sample or concrete blocker.
Unsupported findings are recorded honestly; no missing MVP source is waved through by analogy.

## Phase 0 — Foundation

- Nx + pnpm workspace; `apps/{api,worker,web}` skeletons; `packages/*` with boundary rules.
- `docker-compose` (Postgres) + Zod-validated env config, including separate runtime
  `DATABASE_URL` and unpooled admin/backup `DATABASE_DIRECT_URL` (N-15).
- Drizzle schema v1 + first migration (canonical plan §4 tables).
- Two-tier CI: `dev` push → lint/typecheck/unit/integration (Testcontainers);
  `main` merge → e2e/build.
- Docs live: AGENTS/CLAUDE/CONTEXT, ledger, ADR 0001–0007, prompts, and current board
  (manual — O-07: automation deferred until ≥5 active stories or the first status drift).
- `.kiro/steering/` seeded from CONTEXT.md + canonical plan.
- Separate healthchecks.io checks created for primary cycles and backup workflow (including no-op);
  bot token & keystore secrets in GitHub Secrets.

**Gate:** `pnpm nx run-many -t lint,typecheck,test` green locally and in CI ·
`/api/v1/health` → 200 · empty dashboard renders in LTR **and** RTL (the Phase-0 shell carries
brand/placeholder strings only, served through a seeded EN catalog in `packages/i18n`; the
no-hardcoded-string rule binds from the first real user-facing string, and full EN/TR/AR lands in
Phase 2) · board is current (manual, per O-07).

## Phase 1 — Walking skeleton

One thin slice, no beauty: Telegram `/add <pilot-source-url>` (spike-winning notebook source) →
adapter → preview → approve → target + entry + first observation → scheduler runs
`run-due-checks` → price change → Telegram alert. Backup lane workflow live; per-entry lease
prevents simultaneous primary/backup ownership; per-source fresh/no-op path verified; both
healthchecks pinging. The primary machine is hardened per `runbooks/windows-server.md`
(no sleep, auto-start, restart-on-failure, backup basics — D-29).

**Gate:** a real price row in Neon · a fixture-forced change produces exactly **one** alert ·
a replay produces no new delivery row · deliberately concurrent primary/backup calls claim the
entry once · a delayed worker from an expired/reclaimed lease fails the per-claim-token check and
produces no effects · a `seller_label`-only fixture change produces no semantic observation/event ·
the accepted post-send/pre-commit duplicate window is documented/tested · backup no-op is visible
in `scrape_runs` · both healthchecks show pings · CI green.

## Phase 2 — MVP build-out (strict order)

1. arabam.com adapter (used-car listing URLs).
2. OEM adapters — five brands (O-10): Kia TR (Sportage) + Toyota TR (RAV4) first, then
   VW TR (Tayron), Škoda TR (Kodiaq), BMW TR (X3); PDF-capable, daily cadence.
3. Alert rules (target price, % drop, back-in-stock) + cooldown; rule editor.
4. Dashboard: `dashboard-01` shell · items table · item detail with price-history chart ·
   **cross-site comparison view** (one target ↔ N source entries; linking human-approved) ·
   add flows · settings (AI routes).
5. AI discovery (free text → candidates → approval) + eval set (20–50 real samples) + `/model`.
   Before live calls: exact API credentials/billing recorded; call/token/retry/concurrency limits live.
6. **i18n:** shared `i18next` core catalogs across UI + bot + notifications; React binding only
   in web; fallback/plural/number/date/money tests; untranslated scraped titles; full RTL pass.
7. **PWA:** manifest + service worker (web profile only) + Tailscale Serve HTTPS;
   real-phone Home-Screen install.
8. **Realtime:** thin WS invalidation channel (D-30) — upgrade identity rejection test, 30-minute
   soak over the actual Windows + Tailscale Serve path, forced network-loss reconnect, and polling
   takeover. A failed transport gate ships polling-only and reopens WS; it does not block MVP.

**Gate (MVP exit):** 15+ targets across **all 7 selected MVP sources** (notebook spike winner +
arabam + 5 OEMs) tracked for 2 weeks · alert accuracy ≥ 90% · a change arriving while the
dashboard is open appears without manual refresh when WS passed its gate; otherwise documented
polling-only fallback meets the freshness interval (D-30) ·
nightly live smoke green (or its failure alert demonstrably fired) · locale switch works in all
3 languages **including bot replies and a notification** · PWA installed on a real phone via
Tailscale Serve identity with no browser API secret · phase review done with both prompts +
reconciliation record.

## Phase 3 — Native

Tauri desktop (Windows locally; macOS/Linux via CI) → Android **signed APK** (personal keystore;
`adb install` on a real phone; no store) → iOS **build + simulator smoke only** (CI macOS —
free on the public repo). `tauri-driver` smoke set. Desktop niceties (D-29): tray icon,
scan-now, deep links, backend connection status.

**Gate:** desktop artifacts for 3 OSes in CI · APK completes core flows on a real phone ·
iOS simulator smoke green.

## Phase 4 — Post-MVP / hardening

Category/list-URL tracking epic — including filter-based **SKU/trim discovery** ("new Turkish
SKU appeared" alerts) and **dealer-inventory depth** (trim/dealer/city/color/stock count)
(O-11, D-29) · analytics catalog: 24h/7d/30d min-avg, volatility, seller comparison, deal score
(D-29) · full `Seller` + per-seller source-offer model and new-seller alerts alongside marketplace adapters ·
new/removed listings, pagination caps, removal confirmed after 2 consecutive misses ·
more adapters · Crawlee **if** list crawling demands it ·
sahibinden.com spike (**only if** proxy budget is approved, MVP is stable for 1 month, AND a
documented terms/access review confirms the intended method is permitted/authorized) ·
bot → Cloudflare Workers webhook **if** the D-03 trigger fires (uptime < 95%/mo or external
command need) · backup/restore runbook drill · optional DuckDB analytics experiment.

## Phase 5 — SaaS-readiness

Real auth (pairing/sessions per the P2 design, or OAuth) · quotas/limits · Astro landing
(separate app) · billing research · KVKK/GDPR texts · ToS/legal review + affiliate-API path
before any scraping-as-a-service.

## Phase 6 — Enterprise (on real demand only)

SSO/SCIM · RBAC · audit log · SLA/SLO · DR/backup policy · compliance.
