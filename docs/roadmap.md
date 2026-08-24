# Roadmap

> Phase-based and **date-free** (owner decision O-05). Each phase ends at an **evidence gate**:
> every criterion is answered with proof — command output, screenshot path, DB query result, or
> commit SHA. No evidence, no gate. **Iron rule (D-07):** nothing broadens until Phase 1's first
> real notification works end to end.

## Phase −1 — Risk spike (1–2 days, throwaway code)

Prove the pilots are extractable **before** building any foundation:

- Vatan product page: price/stock via plain fetch + JSON-LD/embedded state?
- ASUS E-Store catalogue JSON endpoint + stable `SalesModelName` selector: extractable? (Notebook
  pilot = preferred candidate with five conditions, Vatan vs ASUS — O-08.)
- arabam.com listing page: price/status extractable? Which ladder rung?
- Every committed OEM endpoint: Kia Sportage, Toyota RAV4, VW Tayron, Škoda Kodiaq, BMW X3.
  Record each endpoint's actual format/rung and one sample; do not assume equivalent source classes.
- Probes use watchlist items where the source carries them; Vatan's off-watchlist sample is an
  extractability control, and the watchlist's lack of a concrete arabam listing means an arbitrary
  live listing is probed (O-09).

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
- Two distinct healthchecks.io checks exist for primary cycles and the backup workflow (including
  valid no-op); bot token & keystore secrets in GitHub Secrets.

**Gate:** `pnpm nx run-many -t lint,typecheck,test` green locally and in CI ·
`/api/v1/health` → 200 · empty dashboard renders in LTR **and** RTL (the Phase-0 shell carries
brand/placeholder strings only, served through a seeded EN catalog in `packages/i18n`; the
no-hardcoded-string rule binds from the first real user-facing string, and full EN/TR/AR lands in
Phase 2) · board is current (manual, per O-07).

## Phase 1 — Walking skeleton

**Entry conditions (from PT-001, O-08/O-19):** Before walking-skeleton work builds on the ASUS
adapter, the four blocking conditions are (a), (c), (d), and (e); condition (b) has a documented
provisional path: `SortPrice` is provisionally the payable price and is mandatorily verified on the
first discounted SKU observed during Phase 1:

- (a) `ShopFilterResult` returns HTTP 200 from a real GitHub Actions runner; a challenge there
  means `unsupported` on that lane, with no workaround.
- (b) The payable-price field is confirmed on a discounted SKU. If no discounted SKU is observable
  when Phase 1 starts, `SortPrice` is provisionally treated as the payable price (sanity-guarded),
  and this condition is mandatorily verified on the first discounted SKU observed during Phase 1.
- (c) Absence semantics follow O-19: require an exhaustive unfiltered response, leave the baseline
  untouched on an inconclusive response, and treat an exhaustive miss as a delisting candidate
  only after a second complete run confirms it.
- (d) Entry representation is a catalogue URL plus a stable `SalesModelName` selector, with the
  `-Gaming` identity confirmed against the owner SKU.
- (e) Exact TRY-to-kuruş bigint parsing is proven across the observed formats.

**Backup-lane entry condition (O-20, PT-005):** before the backup lane's evidence path goes live,
the owner settles how an unvalidatable snapshot artifact stops suppressing the primary health ping.
Until then D-12's suppression rule has no bounded escape, and a single unparseable artifact can hold
the primary dead-man check red for that artifact's remaining lifetime.

One thin slice, no beauty, and it carries change-snapshot capture so Phase-2 alerts are verifiable:
Telegram `/add <pilot-source-url>` (spike-selected preferred notebook source; for the ASUS adapter,
the URL is the catalogue endpoint plus a SKU selector chosen in preview/approve) → adapter →
preview → approve → target + entry + first observation → scheduler runs `run-due-checks` → price
change → Telegram alert. Backup lane workflow live; the url-group fence plus entry effect leases
prevent simultaneous primary/backup ownership; per-source fresh/no-op path verified; both
healthchecks pinging. The primary machine is hardened per `runbooks/windows-server.md` (no sleep,
auto-start, restart-on-failure, backup basics — D-29).

**Gate:** a real price row in Neon plus its initial `accepted` baseline snapshot · a fixture-forced
change produces exactly **one** alert and exactly one new accepted snapshot, forming the verified
before/after pair · an injected primary snapshot-write failure leaves no accepted observation,
baseline move, event, or delivery; an injected post-commit rename failure leaves a protected
`pending` file, suppresses the ping, and reconciles idempotently · one backup execution with changes
in two URL groups uploads one strict multi-envelope artifact with `retention-days: 30`; a no-change
execution uploads none · those backup snapshots are ingested into the local `snapshots/`
store under their terminal lifecycle states before the next primary success ping:
duplicate/out-of-order listings stay idempotent, while a wrong-workflow artifact, digest/envelope
mismatch, extra/archive-traversal path, or failed download is rejected, retried, and suppresses that
ping · an injected backup artifact-upload failure leaves
no accepted observation, baseline move, event, or delivery; a later fresh-fetch retry may finalize
only after its own upload succeeds · a within-budget delayed upload succeeds after pre-upload lease
renewal, while a beyond-TTL candidate produces no effects and still-live candidates finalize once ·
a crash after upload leaves no effects for unfinished candidates; workflow-terminal + lease-expired
reconciliation marks them abandoned/discardable without a permanent healthcheck outage · a
fixture-forced suspicious/held value (>70% swing) fires no
alert and never becomes the baseline, yet writes a `held` snapshot; accept/reject atomically moves
that lifecycle state · with all fixtures older than 35 days, the sweep keeps the two newest
`accepted` snapshots plus every `held`/`pending` file, does not let newer held files displace the
baseline pair, deletes eligible `discardable`/older accepted files, and orders by capture time even
when filesystem `LastWriteTime` is inverted · a replay produces no new delivery row · deliberately
concurrent primary/backup calls claim one URL group once · two entries sharing one catalogue URL are served by
**one** fetch, and the two lanes never fetch that url simultaneously; a third member made due or
inserted while that fetch is in flight cannot open another fetch because `fetch_leases` is live
(O-19) · a delayed worker from an expired/reclaimed group or entry lease fails the token checks and
produces no effects · a `seller_label`-only fixture change produces no semantic observation/event · the
accepted post-send/pre-commit duplicate window is documented/tested · backup no-op is visible in
`scrape_runs` · the primary machine's ops install is proven rather than assumed: `AIPT-primary-up`,
`AIPT-neon-backup` and `AIPT-retention-sweep` each exist with the **expected action, trigger and
principal** (`schtasks /Query /TN … /XML` output, not merely a matching task name) and each one's
smoke run is recorded as succeeded, the pinned ISRG Root X1 passes its SHA-256 check, and one real
`verify-full` Neon dump validates under `pg_restore --list` — the live half PT-004 deferred when the
runbook's ops-doc scope closed · both healthchecks show pings · CI green.

## Phase 2 — MVP build-out (strict order)

1. arabam.com used-car category retained, but adapter **blocked/not built**: direct listing pages
   remain unsupported unless a permitted/authorized first-party route is found. A minimal periodic
   reopen probe (one GET checking `Cf-Mitigated`) is allowed; dropping the category would reopen
   owner-locked D-07 and return the decision to the owner.
2. OEM adapters — five brands (O-10): Kia TR (Sportage) + Toyota TR (RAV4) first, then
   VW TR (Tayron), Škoda TR (Kodiaq), BMW TR (X3); PDF-capable, daily cadence. RAV4 remains the
   target-exists/no-published-price test with a successful run, no observation, and no false
   delisted/out-of-stock event; a temporary priced Corolla Cross canary from the same
   `fiyat_v3.xml` feed proves ordinary extraction and is archived, never deleted, when RAV4 pricing
   returns.
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

**Gate (MVP exit):** 15+ targets across all currently-supported selected sources (preferred
notebook source + 5 OEMs) tracked for 2 weeks (the watchlist is seed data: the owner adds real
targets during the MVP window to reach 15+; the trimmed seed list of 1 notebook SKU + 6 car
targets does not cap this gate); arabam is explicitly recorded as unsupported-blocked, not
silently dropped · on a dual-price fixture, a campaign price disappearing and later reappearing is
held suspicious on **each** flip — no alert, no baseline move, `list_price_minor` preserved — and
an ordinary campaign-value change still alerts on the payable price (D-12) · delivered-alert
precision ≥ 90%: every delivered alert is verified promptly,
at or near delivery time, against the independent contemporaneous record captured by the run that
produced it — the sanitized page snapshot/diagnostic. The stored observation is never ground truth
(that would be circular), and a later live-page fetch is never ground truth (it proves nothing about
the past transition). An alert is correct only when its contemporaneous record supports the claimed
transition; otherwise it is a false positive. Verification happens inside the two-week window,
while the evidence still exists — local snapshots are retained at least 35 days and backup-lane
snapshots live in Actions artifacts with `retention-days: 30` and are validated into the local
store before the next primary success ping, so the before/after chain survives artifact expiry
when primary returns within 30 days; a longer outage is D-12's accepted evidence-loss risk. An alert
whose mandatory snapshot evidence is missing or unusable therefore counts as a **failed
verification**; it is never excluded from the sample, because excluding unverifiable alerts would
let them inflate precision. The window/sample extends until at least 10 delivered alerts have been
scored. Precision = correctly supported delivered alerts ÷ scored delivered alerts. Recall (missed
real changes) is explicitly OUT OF SCOPE of this metric; nightly live-smoke and the RAV4
no-false-event criteria are health checks, not recall measures · a change arriving while the
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
