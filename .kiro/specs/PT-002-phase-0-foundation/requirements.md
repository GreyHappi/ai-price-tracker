# PT-002 — Phase 0 Foundation Requirements

## 1. Purpose and scope

PT-002 establishes only the foundation required by Phase 0 of `docs/roadmap.md`: the pinned
Nx/pnpm workspace, architectural boundaries, PostgreSQL/Drizzle seam, schema v1, health API,
empty internationalized web shell, test harnesses, two-tier CI, security checks, monitoring and
secret provisioning contracts, steering seed, shared Neon URL parser, and executable repository
commands. It does not implement a source adapter, scheduler cycle, notification delivery, backup
evidence ingestion, or any Phase-1 product behavior.

Binding sources for this foundation are grouped as follows:

- architecture, data, runtime, API, and quality: D-01, D-02, D-04 through D-06, and D-08 through
  D-17;
- delivery, review, specification, and repository process: D-18 through D-24, D-26, D-27, and
  D-29;
- the direct-Neon, correctness, and audit-follow-up amendments N-15, N-17, and N-18, plus the
  schema-bearing outcomes O-14, O-18, and O-19.

O-20 remains open and binding as an unresolved owner decision for Phase 1's backup-evidence path;
it does not block T01 or Phase 0. If this document and a canonical source appear to conflict, the
canonical source wins.

## 2. Ubiquitous language

- **Runtime URL** — pooled `DATABASE_URL` used by long-running application traffic.
- **Direct URL** — non-pooler `DATABASE_DIRECT_URL` used for migrations and direct-only
  administration.
- **Entry** — one `source_entries` row: a target-specific URL and optional adapter selector.
- **URL group** — all entries sharing an `adapter_key` and canonical URL.
- **Group fence** — a `fetch_leases` lease for one URL group.
- **Entry lease** — the effect fence stored on a source entry.
- **Snapshot** — a semantic observation; the canonical hash tuple is
  `(price_minor, list_price_minor, currency, in_stock)`.
- **Primary check** — the owner-controlled healthchecks.io dead-man check for successful primary cycles,
  including required backup-snapshot ingestion.
- **Backup check** — the independent healthchecks.io dead-man check for successful backup-workflow
  completion, including a valid no-op.
- **Empty shell** — the Phase-0 web route with no product functionality and with all visible text
  sourced from the real English catalog.
- **Dev tier** — checks required on pushes to `dev`.
- **Main tier** — phase-gate checks required on pushes/merges to `main`.

## 3. Requirements

### R1 — Reproducible workspace and toolchain

**User story:** As a maintainer, I want one reproducible monorepo toolchain so that local and CI
results do not depend on ambient package-manager or generator defaults.

#### Acceptance criteria

- **R1.1** WHEN dependencies are installed, THE FOUNDATION SHALL require Node.js 24 LTS, pin
  `packageManager` to `pnpm@11.23.0`, pin Nx and every `@nx/*` package to the same `23.1.1`
  release, and use a committed `pnpm-lock.yaml` with frozen-lockfile CI installation.
- **R1.2** WHEN Nx discovers the workspace, THE FOUNDATION SHALL expose exactly the Phase-0 project
  names `api`, `worker`, `web`, `contracts`, `domain`, `db`, `scraping`, `ai`, `i18n`,
  `notifications`, and `testing`; no `native` or speculative application SHALL be scaffolded.
- **R1.3** THE FOUNDATION SHALL expose named, directly executable Nx `lint`, `typecheck`, `test`,
  and `build` targets wherever applicable, plus executable `api:e2e`, `web:e2e`,
  `api:openapi-check`, `api:health-smoke`, and `db:migration-check` targets used by the relevant
  gates. A target is explicit when it is present in Nx's RESOLVED project configuration, whether it
  was inferred by a registered plugin or manually declared; `nx show project` and the workspace
  contract test SHALL prove the same required name, executor/configuration, and runnable behavior,
  and a package script that Nx cannot invoke SHALL NOT satisfy this criterion. Concretely, `test`
  SHALL be contributed by `@nx/vitest` 23.1.1 registered as a plugin in `nx.json` with options
  `{ "testTargetName": "test", "testMode": "run" }`, backed by a committed per-project
  `vitest.config.*` (or test-configured `vite.config.*`) in every applicable project; `testMode`
  `run` is required because Vitest defaults to watch mode in an interactive terminal, so only the
  non-watch mode keeps `pnpm nx run-many -t test` and the gates deterministic and terminating
  everywhere. Any target backed by a pinned inference plugin MAY be inferred or manually declared,
  whichever the pinned installation produces: `lint` from `@nx/eslint`, `build` and `typecheck`
  from `@nx/vite` or `@nx/js`, and — because R1.1 pins `@nx/playwright` at the same 23.1.1
  release — `e2e` from the `@nx/playwright` plugin's default `e2e` target name, which is the route
  available to `web:e2e`. Any target with NO pinned inference provider SHALL be manually declared:
  `api:openapi-check`, `api:health-smoke`, `db:migration-check`, the `testing:compose-smoke`,
  `testing:neon-parser-check`, `testing:secret-scan`, and `testing:markdown-check` gates, and
  `api:e2e`, which is a Nest/Testcontainers suite that the `test`-named `@nx/vitest` registration
  does not contribute. Neither route is privileged; the criterion above — present in
  `nx show project` output, asserted and invocable through the workspace contract test — remains
  the only binding one, and Design D3 carries the per-target classification.
- **R1.4** WHEN generators are used, THE FOUNDATION SHALL explicitly select Vite/Vitest or no
  generated test runner as appropriate and SHALL contain no Jest configuration, dependency, or
  generated Jest test.
- **R1.5** THE FOUNDATION SHALL use flat ESLint configuration and SHALL treat warnings as failures
  in repository gates.

### R2 — Enforced architecture boundaries

**User story:** As a maintainer, I want illegal dependencies rejected mechanically so that the lean
modular-monolith shape does not drift.

#### Acceptance criteria

- **R2.1** THE FOUNDATION SHALL assign a stable project type and scope tag to every project and
  SHALL enforce the complete dependency matrix in Design D2 with Nx's flat-config
  `@nx/enforce-module-boundaries` rule.
- **R2.2** `domain` SHALL have no workspace dependency, application projects SHALL NOT import one
  another, production projects SHALL NOT import `testing`, and ADR-0004's `db` package SHALL depend
  on `contracts` only, never `domain` or another workspace project.
- **R2.3** `i18n` SHALL be framework-neutral and SHALL NOT depend on or export React,
  `react-i18next`, browser globals, or a web component.
- **R2.4** `api`, `worker`, and all backend packages SHALL reject imports of React, React DOM,
  React i18n bindings, and `web`, even if a future tag edit would otherwise allow one.
- **R2.5** WHEN a forbidden-edge fixture is linted, THE FOUNDATION SHALL produce a failing lint
  result, proving that the boundary policy is executable rather than documentary.

### R3 — Validated environment and safe local PostgreSQL

**User story:** As an operator, I want pooled runtime access separated from direct migration access
so that an invalid Neon URL fails safely before database work starts.

#### Acceptance criteria

- **R3.1** WHEN configuration is loaded, THE FOUNDATION SHALL validate it with Zod from an
  injectable key/value source and SHALL throw a typed validation error containing field-level
  issues; reusable configuration code SHALL NOT call `process.exit`.
- **R3.2** `DATABASE_URL` and `DATABASE_DIRECT_URL` SHALL be absolute parsed URLs that accept only
  `postgres:` and `postgresql:` schemes and contain a nonempty hostname, database path, username,
  and password; encoded credentials and IPv6 hosts SHALL remain valid inputs.
- **R3.3** IF the normalized hostname of `DATABASE_DIRECT_URL` contains the Neon `-pooler.` marker,
  THE FOUNDATION SHALL reject it; query parameters alone SHALL NOT be used to decide whether a URL
  is pooled.
- **R3.4** WHEN `api` or `worker` starts, and WHEN CI validates environment fixtures, THE
  FOUNDATION SHALL execute the direct-URL assertion before database migration or runtime setup.
- **R3.5** THE local Compose service SHALL use PostgreSQL major 17, bind its published database
  port only to `127.0.0.1`, omit the obsolete top-level Compose `version`, and obtain credentials
  from a gitignored `.env` without a credential literal in tracked Compose. A tracked
  `.env.example` MAY contain only unmistakably dummy local values; it SHALL contain no production
  or Neon secret. An automated smoke target SHALL start the service with those dummy values, wait
  for readiness, execute a real `SELECT 1`, and always tear the stack down.
- **R3.6** THE FOUNDATION SHALL commit only safe example values and variable names; real secrets,
  production URLs, and credentials SHALL remain absent from tracked files and generated output.

### R4 — Canonical PostgreSQL schema v1

**User story:** As a later story implementer, I want the canonical storage seams established once so
that Phase-1 behavior does not need a foundational rename or relationship rewrite.

#### Acceptance criteria

- **R4.1** THE schema SHALL define exactly these canonical tables:
  `workspaces`, `users`, `tracking_targets`, `source_entries`, `fetch_leases`, `observations`,
  `change_events`, `notification_deliveries`, `scrape_runs`, `ai_calls`, `app_settings`,
  `discovery_requests`, and `discovery_candidates`; it SHALL NOT add a target-entry join table.
- **R4.2** THE relation from `tracking_targets` to `source_entries` SHALL be one-to-many through
  `source_entries.target_id`; `workspace_id` SHALL appear only on aggregate roots, and child rows
  SHALL reach a workspace through their parent.
- **R4.3** `tracking_targets.archived_at` SHALL be the soft-archive seam. THE schema SHALL NOT add a
  database snapshot lifecycle/status column; the lifecycle defined by D-12 remains encoded in
  backup artifacts and protocol state outside the observation row. Tracking/history foreign keys
  SHALL use `NO ACTION`/`RESTRICT`, not cascade deletion.
- **R4.4** `source_entries` SHALL carry target, site/adapter, URL, nullable selector, currency,
  health/scheduling fields, and the entry-lease owner/until pair. It SHALL enforce
  `UNIQUE (target_id, url, selector) NULLS NOT DISTINCT` while allowing different targets or
  selectors to share a URL.
- **R4.5** `fetch_leases` SHALL use `(adapter_key, canonical_url)` as its composite primary key.
  Group and entry claims SHALL use a fresh UUID fencing token for every claim, never a stable
  worker/lane identifier, and all due/fresh/lease comparisons SHALL use PostgreSQL `now()`.
- **R4.6** `observations` SHALL be append-only semantic snapshots with entry and previous-observation
  links, both money fields, currency, nullable `in_stock`, nullable `seller_label`, extraction
  method/confidence, the exact snapshot hash tuple from the glossary, and UTC observation time. It
  SHALL enforce `UNIQUE (entry_id, prev_observation_id) NULLS NOT DISTINCT` and SHALL contain no
  raw response body.
- **R4.7** `change_events.observation_id` SHALL be unique. `notification_deliveries` SHALL have a
  unique `dedupe_key`, attempt count, lease, and constrained `pending -> sending -> sent|failed`
  outbox states that preserve D-05 at-least-once semantics. The documented `dedupe_key` identity
  SHALL be deterministic from the canonical event, channel, recipient, and future rule ID once
  rules exist—never a timestamp, attempt counter, or ID minted by a retry; runtime derivation
  remains with the notification-owning phase.
- **R4.8** `scrape_runs` SHALL record run kind, nullable entry, adapter, lane, status, start time,
  duration, and error class; it SHALL enforce `entry_id IS NOT NULL OR kind = 'probe'` and SHALL
  store neither raw bodies nor unbounded diagnostic content.
- **R4.9** Money SHALL be signed 64-bit integer minor units with positive-value checks where a price
  is present. Currency SHALL use one non-padding `varchar(3)` representation with an uppercase
  ISO-shape check. All persisted instants SHALL use UTC-capable `timestamptz`; `in_stock` SHALL be
  nullable to preserve the unknown state.
- **R4.10** Database `CHECK` constraints and the corresponding shared Zod contracts SHALL agree on
  nullable states, positive-money rules, currency shape, and cross-column invariants. Exact
  canonical sets SHALL be limited to locales `en|tr|ar`, lanes `primary|backup`, and outbox statuses
  `pending|sending|sent|failed`; `probe` and `ai` SHALL be accepted reserved literals for run kind
  and extraction method. Canonically unnamed source health, target/change kinds, ordinary run kinds,
  run terminal statuses, error classes, and channels SHALL use aligned bounded nonempty seams rather
  than invented exhaustive enums. JSON bodies SHALL serialize bigint money as decimal strings.
- **R4.11** `ai_calls`, `app_settings`, `discovery_requests`, and `discovery_candidates` SHALL expose
  only the minimum canonical audit/config/discovery seams in Design D4; their Phase-2 behavior,
  AI routing defaults, and candidate workflow SHALL remain deferred.
- **R4.12** THE initial migration SHALL deterministically and idempotently create exactly one MVP
  seed workspace using one fixed application identifier. Reapplying migrations SHALL preserve that
  single row; user, Telegram-chat, settings, and other feature seeds SHALL NOT be invented.

### R5 — Drizzle runtime and migration seam

**User story:** As a developer, I want migrations and runtime access to share one schema while using
the correct connection mode.

#### Acceptance criteria

- **R5.1** THE FOUNDATION SHALL configure current Drizzle Kit PostgreSQL generation and migrations
  (`drizzle-kit generate` and `drizzle-kit migrate`) against the direct URL and SHALL make schema
  drift an executable failing check.
- **R5.2** THE migration entry point SHALL accept an injected direct connection or connection
  factory so a Testcontainers PostgreSQL instance can exercise it without mutating global process
  state.
- **R5.3** THE runtime database factory/provider SHALL use the validated pooled runtime URL and
  SHALL be consumable by both `api` and `worker` without either project defining a second schema.
- **R5.4** WHEN migrations run twice against the same empty test database, THE second run SHALL be
  a successful no-op, every constraint in R4 SHALL remain present, and exactly the same one seed
  workspace SHALL exist.

### R6 — Contract-first health API

**User story:** As an operator, I want one honest loopback health endpoint so that a successful
response proves the API can query PostgreSQL.

#### Acceptance criteria

- **R6.1** THE canonical healthy and unhealthy response Zod 4 schemas SHALL live in `contracts`; the API SHALL
  use the locked `nestjs-zod` integration, and Nest DTO/OpenAPI metadata SHALL be derived from
  those schemas rather than maintained as parallel handwritten types.
- **R6.2** THE API SHALL set global prefix `api`, enable URI versioning with default version `1`, and
  mount a controller path of only `health`, yielding exactly `/api/v1/health` without a hardcoded
  `api/v1` duplicate.
- **R6.3** THE API SHALL listen on `127.0.0.1` by default.
- **R6.4** WHEN the Drizzle provider can execute a minimal database probe, GET `/api/v1/health`
  SHALL return HTTP 200 with exactly `{ "status": "ok", "database": "up" }`; IF PostgreSQL is
  unavailable, it SHALL return HTTP 503 with exactly
  `{ "status": "error", "database": "down" }` without exposing credentials. A custom Nest
  controller SHALL own that response, mapping probe success to HTTP 200 and a database exception to
  a sanitized HTTP 503 whose body IS that exact flat unhealthy variant of the `contracts` health
  schema and nothing else — no envelope, wrapper, or additional `error`, `message`, `statusCode`,
  or other key — and THE health endpoint SHALL NOT use the API's global shared error shape.
  Terminus SHALL NOT appear in the health path or the dependency set. The injected Drizzle probe
  seam SHALL remain.
- **R6.5** THE checked-in OpenAPI snapshot SHALL be produced from the running Nest application and
  shared Zod DTOs, passed through `nestjs-zod`'s required `cleanupOpenApiDoc`, and CI SHALL fail on
  an uncommitted contract delta.

### R7 — Internationalized, RTL-safe empty shell

**User story:** As a future user, I want localization and directionality built into the first screen
so that later UI work does not retrofit them.

#### Acceptance criteria

- **R7.1** `packages/i18n` SHALL initialize framework-neutral i18next resources from a real
  file-based English catalog and SHALL own locale fallback, interpolation/plural rules, and
  number/date/bigint-money formatting; visible shell strings SHALL NOT be inline literals.
- **R7.2** React binding and direction synchronization SHALL live under `web`, and changing the
  active direction SHALL set document `lang` and `dir` consistently.
- **R7.3** THE Phase-0 shell SHALL render the checked-in English catalog in normal LTR and forced RTL
  modes. The Phase-0 roadmap gate explicitly requires only that English shell; complete Turkish
  and Arabic catalogs SHALL remain a Phase-2 deliverable under D-23's binding EN/TR/AR commitment.
- **R7.4** Flat ESLint SHALL enable an installed, real hardcoded-user-string rule compatible with
  the configuration (`eslint-plugin-i18next` / `i18next/no-literal-string`) in officially supported
  `mode: 'jsx-only'`, with only narrow structural JSX-attribute exclusions. It SHALL fail controlled
  literals in both visible JSX text and user-visible `aria-label`/`placeholder` attributes.
- **R7.5** Vitest/React Testing Library SHALL assert semantic rendering and direction attributes;
  framework-neutral tests SHALL explicitly cover English fallback, pluralization, and formatting;
  only Playwright in a real browser SHALL assert RTL geometry, overflow, and overlap.

### R8 — Proportionate test harnesses

**User story:** As an implementer, I want the right test environment for each risk so that passing
tests mean more than mocked success.

#### Acceptance criteria

- **R8.1** Backend and frontend unit tests SHALL use Vitest; Jest SHALL not be present.
- **R8.2** Database integration and API e2e tests SHALL use a real PostgreSQL 17 Testcontainers
  instance and SHALL NOT replace migration, constraint, or health-probe behavior with an in-memory
  database.
- **R8.3** Web component tests SHALL use React Testing Library and MSW for HTTP boundaries, while
  layout and RTL geometry SHALL run through the invoked `web:e2e` Playwright target. Local and CI
  setup SHALL install the pinned Chromium browser binary before that target runs.
- **R8.4** `fast-check` SHALL be limited to high-value invariants such as URL classification and
  money/currency validation; THE FOUNDATION SHALL NOT impose a repo-wide property or coverage gate.
- **R8.5** Coverage thresholds SHALL be introduced only for high-value pure domain/parser logic
  when such logic exists, not for the Phase-0 repository as a whole.

### R9 — Two-tier CI and security gates

**User story:** As a maintainer, I want branch-aligned gates so that `dev` stays fast and `main`
receives phase-gate scrutiny.

#### Acceptance criteria

- **R9.1** ON a push to `dev`, CI SHALL install from the frozen lockfile and run lint, explicit
  typecheck, unit tests, integration tests including the Testcontainers database suite, and the
  pinned Gitleaks history/working-tree scan.
- **R9.2** ON a push or merge to `main`, CI SHALL run the dev-tier checks plus the Compose smoke,
  Neon parser self-test, invoked API/web e2e, OpenAPI snapshot, migration-drift, and
  `api`/`worker`/`web` production builds. CI SHALL provision ephemeral PostgreSQL 17 connection
  values for checks that require `DATABASE_URL` or `DATABASE_DIRECT_URL`, and SHALL install the
  pinned Chromium binary before `web:e2e`. Publishing, signing, packaging, and release targets
  SHALL remain deferred; no elapsed-time SLO SHALL be invented for either tier.
- **R9.3** GitHub workflow actions SHALL be current supported releases, use `pnpm/setup@v2` for
  pnpm 11/Node 24 provisioning, and be pinned to immutable full commit SHAs with the release tag
  recorded in a comment. Its inputs SHALL pin `runtime: node@24` and `install: false`, followed by
  exactly one explicit `pnpm install --frozen-lockfile`; `pnpm/action-setup` SHALL NOT be used with
  pnpm 11.
- **R9.4** THE repository SHALL run a real pinned Gitleaks scan of Git history and the working tree
  using current `gitleaks git` and `gitleaks dir` commands; deprecated or no-op scan commands SHALL
  not satisfy the gate.
- **R9.5** Owner-controlled GitHub secret scanning and push protection SHALL be enabled and recorded
  as external evidence; test fixtures SHALL contain only recognized dummy markers.
- **R9.6** Phase-0 check/build workflows SHALL declare least-privilege
  `permissions: contents: read` at workflow or job scope. Any future additional permission SHALL be
  narrowly scoped and justified by an actually invoked operation; Phase 0 requires no write grant.

### R10 — Monitoring and secret provisioning contract

**User story:** As an operator, I want independent dead-man checks and correctly scoped secrets
ready before cycles exist so that Phase 1 can wire runtime pings without conflating failure modes.

#### Acceptance criteria

- **R10.1** THE owner-controlled Phase-0 provisioning task SHALL ensure that two distinct
  healthchecks.io checks exist: one primary-cycle check and one backup-workflow check. It SHALL
  reuse and identify a correctly configured existing check or create the missing check, without
  renaming either after Windows Scheduled Tasks such as `AIPT-primary-up` or
  `AIPT-neon-backup`. THE recorded evidence SHALL state which path applied per check — identified
  or created — together with each check's distinct redacted identifier and its role assignment,
  either primary-cycle or backup-workflow-including-valid-no-op. IF more than two candidate checks
  exist, or a role is ambiguous, THE task SHALL stop and the owner SHALL designate the canonical
  pair. It SHALL NOT invent cadence or grace values absent from owner configuration; missing values
  remain incomplete owner evidence rather than guessed configuration.
- **R10.2** GitHub Secrets SHALL contain distinct placeholders/credentials for exactly six names:
  `TELEGRAM_BOT_TOKEN`, `HEALTHCHECKS_IO_BACKUP_KEY`, and the future Android signing set
  `ANDROID_KEYSTORE_BASE64`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, and
  `ANDROID_KEY_PASSWORD`; no real value SHALL appear in documentation, examples, logs, or
  repository files. `HEALTHCHECKS_IO_PRIMARY_KEY` SHALL NOT be among them: no Phase-0 or Phase-1
  GitHub-hosted consumer reads it, so it SHALL live solely in the primary server's gitignored
  `.env`. Re-adding it SHALL require the same change that amends D-02 to legitimize a GitHub-hosted
  primary consumer.
- **R10.3** A public-repository backup workflow SHALL receive only the backup healthcheck credential;
  it SHALL NOT receive or reference the primary credential.
- **R10.4** Because Phase 0 has no scheduler cycle or backup-lane execution, THE FOUNDATION SHALL
  define provisioning/configuration and secret-scope contracts only. Runtime ping code and the
  D-15 HTTP-wrapper call site SHALL be added with the Phase-1 cycle integration, not prematurely.
- **R10.5** O-20 SHALL remain open. PT-002 SHALL neither add a third evidence check nor define an
  auto-skip, quarantine, retry bound, or other resolution that silently amends D-02 or D-12.

### R11 — Repository-operability contracts

**User story:** As a future AI season or Windows operator, I want one-source operational guidance so
that commands and Neon URL parsing do not drift between copies.

#### Acceptance criteria

- **R11.1** `.kiro/steering/` SHALL be seeded with concise repository guidance that links to
  `CONTEXT.md`, the canonical plan, ledger, active board, and agent contract as sources of truth; it
  SHALL NOT copy arbitrary leading lines or create an independently maintained decision summary.
  A non-mutating Markdown target SHALL validate the Phase-0-touched Markdown, internal links, and
  code fences and SHALL run again after closing documentation changes.
- **R11.2** A shared `parse-neon-url.ps1` SHALL parse and validate the supported Neon PostgreSQL URL
  into the libpq values required by the Windows runbook without printing the original URL or
  password. Its cross-platform Nx self-test target SHALL run in the complete Phase-0 gate while
  retaining Windows PowerShell compatibility.
- **R11.3** The runbook's saved backup script and section 5 audit snippet SHALL dot-source that same
  parser; duplicated parsing implementations SHALL be removed. The runbook setup SHALL copy the
  committed parser to `C:\ops\ai-price-tracker\parse-neon-url.ps1` and assert that the installed
  file exists before either caller can run.
- **R11.4** AFTER executable targets exist, the `AGENTS.md` Commands section SHALL replace its Phase-0
  placeholder with verified pnpm/Nx/Docker/test commands and SHALL link rather than duplicate
  canonical policy.
- **R11.5** The current board SHALL be updated only when Phase-0 evidence exists; already-live
  canonical documentation SHALL not be regenerated merely to satisfy PT-002.

### R12 — Phase-0 completion and deferrals

**User story:** As the owner, I want an evidence-backed phase gate so that Phase 1 starts from a
working foundation without pulling later behavior forward.

#### Acceptance criteria

- **R12.1** WHEN the Phase-0 gate is run locally and on the applicable CI tier,
  `pnpm nx run-many -t lint,typecheck,test` SHALL exit successfully for the complete workspace.
- **R12.2** WITH PostgreSQL available, an assertion against `http://127.0.0.1:3000/api/v1/health`
  SHALL observe HTTP 200; WITH PostgreSQL unavailable, the API e2e suite SHALL observe HTTP 503.
- **R12.3** Playwright SHALL prove the empty shell renders from the English catalog in LTR and forced
  RTL without horizontal overflow or overlap at the committed Phase-0 viewports.
- **R12.4** THE phase record SHALL include green dev/main gate evidence, schema/constraint,
  Compose readiness/SQL, Neon parser, and secret-scan evidence, plus owner-controlled evidence for
  healthcheck provisioning, GitHub secret names/scopes, secret scanning/push protection, board
  state, and D-19's reconciled fresh-session phase reviews.
- **R12.5** Source adapters, `run-due-checks`, lease-claim algorithms, snapshots and alerts,
  healthcheck ping calls, backup evidence ingestion, full TR/AR catalogs, authentication behavior,
  PWA/native deliverables, and all product UI SHALL remain deferred to their canonical phases.

## 4. Requirement-to-design map

| Requirement | Design section |
|---|---|
| R1 | D1, D3 |
| R2 | D2 |
| R3 | D5 |
| R4 | D4 |
| R5 | D5 |
| R6 | D6 |
| R7 | D7 |
| R8 | D8 |
| R9 | D3, D9 |
| R10 | D9 |
| R11 | D10 |
| R12 | D1, D8, D11 |
