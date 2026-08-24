# PT-002 — Phase 0 Foundation Design

## D1. Scope and architecture

This design realizes the Phase-0 roadmap gate and the requirements in `requirements.md`. The
foundation is a lean Nx modular monolith: applications own composition and transport; packages own
reusable contracts, domain concepts, infrastructure adapters, and test support.

```text
web ───────────────> contracts, i18n
api ───────────────> contracts, domain, db, scraping, ai, i18n, notifications
worker ────────────> contracts, domain, db, scraping, ai, notifications
                         │
packages ───────────────┴──> dependency matrix D2
PostgreSQL 17 <──────── db runtime provider / direct migration adapter
```

There are exactly three Phase-0 applications: `apps/api`, `apps/worker`, and `apps/web`. Reusable
projects are exactly `packages/contracts`, `packages/domain`, `packages/db`,
`packages/scraping`, `packages/ai`, `packages/i18n`, `packages/notifications`, and
`packages/testing`. The bot remains a future module inside `api`; no bot app, native app, queue,
adapter implementation, scheduler, healthcheck pinger, or product screen is created here.

This section implements R1 and R12.

## D2. Project boundaries

Every project has one `type:*` tag and one `scope:*` tag. The allow-list below is complete; a blank
cell means no workspace imports. External dependencies are additionally restricted where noted.

| Project | Type | Allowed production workspace imports |
|---|---|---|
| `domain` | library | none |
| `contracts` | library | `domain` |
| `i18n` | library | none |
| `db` | library | `contracts` only |
| `scraping` | library | `domain`, `contracts` |
| `ai` | library | `domain`, `contracts` |
| `notifications` | library | `domain`, `contracts`, `i18n` |
| `testing` | test support | any project, but only from test/config files |
| `api` | application | `domain`, `contracts`, `db`, `scraping`, `ai`, `i18n`, `notifications` |
| `worker` | application | `domain`, `contracts`, `db`, `scraping`, `ai`, `notifications` |
| `web` | application | `contracts`, `i18n` |

The flat root ESLint policy uses `@nx/enforce-module-boundaries` `depConstraints` for this matrix,
forbids app-to-app imports, pins ADR-0004's `db -> contracts`-only edge, and forbids production
imports of `testing`. A separate
`no-restricted-imports` policy rejects `react`, `react-dom`, `react-i18next`, their subpaths, and
the `web` project from `api`, `worker`, `domain`, `db`, `scraping`, `ai`, `notifications`, and
`i18n`. The `i18n` public surface exports only framework-neutral catalog/types/initialization.

Policy tests use controlled invalid fixtures and the ESLint API, expecting lint failure. They do
not leave an intentionally broken source file inside an ordinary project target.

This section implements R2.

## D3. Toolchain and target model

### Verified baseline

The baseline was checked against official upstream sources on 2026-08-24.

| Concern | Phase-0 choice | Binding rule |
|---|---|---|
| JavaScript runtime | Node.js 24 LTS | root engine and CI runtime |
| Package manager | pnpm 11.23.0 | exact `packageManager` pin and committed lockfile |
| Monorepo | Nx 23.1.1 | exact, matching version for Nx and all `@nx/*` plugins |
| CI bootstrap | `pnpm/setup@v2` | immutable full SHA with `runtime: node@24` and `install: false`; the workflow then performs one explicit frozen install, so `pnpm/action-setup@v6`, a redundant normal-path `setup-node`, and the action's default install are excluded |
| Checkout | `actions/checkout@v7` | immutable full SHA with the human-readable tag in a comment |
| Database migrations | Drizzle Kit current command surface | `generate` and `migrate`, PostgreSQL dialect |
| Validation/OpenAPI | Zod 4 + `nestjs-zod` 5.5.0 | shared schemas -> derived DTOs -> cleaned OpenAPI document |
| Lint | ESLint flat config | Nx boundary rule plus `eslint-plugin-i18next` flat rule |
| Tests | Vitest, Testcontainers, RTL/MSW, Playwright | environment selected by risk, not one universal runner |
| Secret scan | Gitleaks 8.30.1 | pinned binary/container; current `git` and `dir` commands |

The package manifest and lockfile are authoritative for exact JavaScript dependency versions. Each
CI job invokes exactly one `pnpm install --frozen-lockfile` after the setup action is told not to
install. Generator calls explicitly disable Jest/default e2e output; Vitest and Playwright
configurations are added intentionally. Nx project names are fixed rather than inferred from
directory basenames that could later change.

| Project class | Pinned generator choices |
|---|---|
| `api` | Nest application; ESLint; no generated unit/e2e runner; node test environment is added later with `@nx/vitest:configuration` |
| `worker` | Node application; ESLint; no generated unit/e2e runner; node test environment is added later with `@nx/vitest:configuration` |
| `web` | React application; Vite bundler; CSS; Vitest; no generated e2e runner or router; Playwright is added later with `@nx/playwright:configuration --project=web` |
| packages | TypeScript Nx libraries with explicit build/lint/typecheck entry points and no generated test runner; Vitest is added deliberately where code exists |

The implementer must use the Nx 23 generator option names shown by that pinned installation and
record the resolved noninteractive commands in the task evidence; accepting an interactive default
is not equivalent to these choices.

### Target contract

- Every production project has an explicit `lint` and `typecheck` target.
- Libraries and applications with runnable code have `test`; both frontend and backend use Vitest.
- `db:test` includes PostgreSQL Testcontainers integration tests and therefore belongs in the dev
  `test` gate; “test” is not constrained to mocked unit tests.
- `api:e2e` starts a Nest test application against PostgreSQL 17 Testcontainers.
- `api:health-smoke` starts a real loopback listener, asserts HTTP 200 without a production global
  `fetch` call, and always tears the listener down.
- `web:e2e` invokes Playwright and owns browser layout/RTL assertions.
- `api:openapi-check` generates the document in a deterministic test profile and fails on snapshot
  delta.
- `db:migration-check` runs Drizzle generation into a disposable comparison location and fails on
  schema/migration drift without rewriting the checked-in migration during verification.
- Build targets produce `api`, `worker`, and `web` production artifacts. Native/release-matrix
  targets do not exist until their roadmap phase.

Official verification sources: [Node releases](https://nodejs.org/en/about/previous-releases),
[pnpm releases](https://github.com/pnpm/pnpm/releases),
[`pnpm/setup`](https://github.com/pnpm/setup),
[Nx releases](https://github.com/nrwl/nx/releases),
[checkout action](https://github.com/actions/checkout),
[Drizzle Kit generate](https://orm.drizzle.team/docs/drizzle-kit-generate),
[Drizzle Kit migrate](https://orm.drizzle.team/docs/drizzle-kit-migrate),
[Nx module boundaries](https://nx.dev/docs/features/enforce-module-boundaries), and
[Nx Playwright configuration](https://nx.dev/technologies/test-tools/playwright/introduction),
[Nest URI versioning](https://docs.nestjs.com/techniques/versioning),
[Nest Terminus](https://docs.nestjs.com/recipes/terminus),
[`nestjs-zod`](https://github.com/BenLorantfy/nestjs-zod),
[`eslint-plugin-i18next` rule options](https://github.com/edvardchen/eslint-plugin-i18next/blob/master/docs/rules/no-literal-string.md), and
[Gitleaks](https://github.com/gitleaks/gitleaks).

This section implements R1 and R9.

## D4. Schema v1 data delta

### Shared storage rules

Schema declarations live in `packages/db`; shared external representations and validation live in
`packages/contracts`. Every entity table except `fetch_leases` uses a UUID primary key. Foreign keys
are explicit and indexed. Application tracking/history FKs use PostgreSQL `NO ACTION`/`RESTRICT`;
none cascades a hard delete through retained history. Persisted instants use `timestamptz`;
application code treats them as UTC. Money is PostgreSQL `bigint` minor units and crosses JSON as a
decimal string. Every currency column is non-padding `varchar(3)` with a `^[A-Z]{3}$` database check
and the same Zod rule. Prices, token counts, costs, attempts, and durations use non-floating integer
storage with appropriate non-negative/positive checks. Defaults and lease comparisons use database
`now()`, never a worker clock.

The Phase-0 aggregate roots carrying `workspace_id` are `users`, `tracking_targets`,
`app_settings`, and `discovery_requests`. Their children do not duplicate it. `workspaces` is the
workspace root itself; infrastructure audit rows that are not workspace aggregates do not gain a
speculative workspace column.

### Tables and binding columns

The following is the migration contract, not an implementation body. Common UUID identifiers and
audit timestamps are included only where needed for identity/FKs/audit; feature columns not listed
remain deferred.

| Table | Binding columns and relationships | Binding constraints/meaning |
|---|---|---|
| `workspaces` | `id`, `created_at` | initial migration inserts one fixed application UUID exactly once; no feature seed or membership/RBAC model |
| `users` | `id`, `workspace_id`, `locale`, `telegram_chat_id`, `created_at` | `workspace_id -> workspaces.id`; locale exactly `en`, `tr`, or `ar`; nullable Telegram identity until configured |
| `tracking_targets` | `id`, `workspace_id`, `kind`, `title`, `target_price_minor`, `archived_at`, `created_at`, `updated_at` | aggregate root; kind is a bounded nonempty seam, not an invented enum; nullable positive target price; soft archive only through `archived_at` |
| `source_entries` | `id`, `target_id`, `site`, `url`, `selector`, `adapter_key`, `currency`, `health`, `last_checked_at`, `next_check_at`, `lease_owner`, `lease_until`, `created_at`, `updated_at` | `target_id -> tracking_targets.id`; `selector` nullable; `UNIQUE (target_id, url, selector) NULLS NOT DISTINCT`; no `UNIQUE(url)`; one currency per entry; health is bounded nonempty text; lease owner/until both null or both present |
| `fetch_leases` | `adapter_key`, `canonical_url`, `lease_owner`, `lease_until` | composite primary key `(adapter_key, canonical_url)`; token is a fresh UUID per claim |
| `observations` | `id`, `entry_id`, `prev_observation_id`, `price_minor`, `list_price_minor`, `currency`, `in_stock`, `seller_label`, `snapshot_hash`, `extraction_method`, `confidence`, `observed_at` | entry and self FKs; positive price and positive nullable list price; nullable tri-state stock; extraction method is bounded nonempty text and must admit reserved `ai`; `UNIQUE (entry_id, prev_observation_id) NULLS NOT DISTINCT`; append-only semantic row; no lifecycle/status or raw body |
| `change_events` | `id`, `observation_id`, `kind`, `created_at` | unique FK `observation_id -> observations.id`; kind is bounded nonempty and extensible, without an invented Phase-0 enum or pre-added Phase-4 values |
| `notification_deliveries` | `id`, `change_event_id`, `recipient_id`, `dedupe_key`, `channel`, `status`, `lease_until`, `attempts`, `sent_at`, `created_at`, `updated_at` | FKs to event/user; channel is bounded nonempty; unique `dedupe_key`; states exactly `pending`, `sending`, `sent`, `failed`; sending owns an expiring lease; attempts non-negative |
| `scrape_runs` | `id`, `entry_id`, `kind`, `adapter_key`, `lane`, `status`, `started_at`, `duration`, `error_class` | nullable FK entry; lane exactly `primary`/`backup`; kind and status are bounded nonempty seams, with `probe` the reserved probe-kind literal; nullable error class is bounded when present; non-negative integer duration; `CHECK (entry_id IS NOT NULL OR kind = 'probe')`; no raw body |
| `ai_calls` | `id`, `provider`, `model`, `purpose`, `tokens`, `cost`, `currency`, `latency`, `ok`, `created_at` | non-negative integer tokens/cost/latency; cost uses minor-unit currency pair; audit seam only, no provider routing logic |
| `app_settings` | `id`, `workspace_id`, `ai_routes`, `created_at`, `updated_at` | workspace aggregate root; `ai_routes` is the DB configuration seam, with Phase-2 values/defaults deferred |
| `discovery_requests` | `id`, `workspace_id`, `created_at` | workspace aggregate root and AI add-flow request seam; request behavior deferred |
| `discovery_candidates` | `id`, `request_id`, `created_at` | child FK to `discovery_requests`; candidate payload/workflow fields deferred to the owning feature spec |

The canonical one-to-many relation is `tracking_targets.id -> source_entries.target_id`; there is no
join table. The source entry persists the adapter-emitted canonical fetch URL as `url`; Phase 0 does
not implement a global URL normalizer.

### Semantic and concurrency invariants

- Snapshot hashing covers exactly `(price_minor, list_price_minor, currency, in_stock)` in that
  order. `seller_label`, timestamps, confidence, attempts, and row IDs never enter the hash.
- `in_stock = null` is different from true and false. `list_price_minor = null` is part of the hash,
  not an omitted field.
- Observation-chain and event uniqueness provide replay seams, but Phase 0 does not implement the
  Phase-1 write algorithm or invent an observation lifecycle column.
- The initial migration inserts one deterministic seed-workspace UUID with conflict-safe semantics.
  A second migration run leaves exactly that one workspace and still creates no user, chat,
  app-settings, or other feature seed.
- A claim produces one fresh UUID, writes it to the group fence and all due member entries, and
  checks both tokens plus both expiries at effect time. The actual claim/renew/finalize transaction
  and two-lane concurrency tests belong to Phase 1; schema columns and constraints belong here.
- Outbox state transitions are `pending -> sending -> sent|failed`. The unique dedupe key prevents
  duplicate enqueue; it does not claim exactly-once Telegram delivery. Its stable identity inputs
  are the canonical `change_event_id`, channel, recipient, and future rule ID once Phase-2 rules
  exist. Timestamp, attempt count, and any retry-minted ID are forbidden inputs. Phase 0 stores and
  constrains the seam but does not implement enqueue or invent the deferred rule model.
- Exact constrained sets are only `users.locale = en|tr|ar`, `scrape_runs.lane = primary|backup`,
  and the four outbox statuses. `scrape_runs.kind` must admit reserved `probe`, and
  `observations.extraction_method` must admit reserved `ai`. Source health, target/change kinds,
  ordinary run kinds, run terminal statuses, error classes, and channels use the same bounded,
  trimmed, nonempty contract in Zod and PostgreSQL rather than speculative exhaustive enums.
  Phase-4 `change_events.kind` values are not added early.
- Neon contains no page body, HTML, snapshot envelope, diagnostic body, or artifact lifecycle
  state. D-12 filesystem/artifact protocols remain outside this migration.

This section implements R4.

## D5. Environment, Compose, Drizzle, and migrations

### Interfaces

TypeScript signatures define the seams; implementations remain in the owning packages.

```ts
type EnvironmentSource = Readonly<Record<string, string | undefined>>;

interface EnvironmentIssue {
  readonly path: readonly (string | number)[];
  readonly message: string;
}

class EnvironmentValidationError extends Error {
  readonly issues: readonly EnvironmentIssue[];
}

interface AppEnvironment {
  readonly databaseUrl: string;
  readonly databaseDirectUrl: string;
  readonly host: '127.0.0.1';
  readonly port: number;
}

function loadEnvironment(source: EnvironmentSource): AppEnvironment;
function parsePostgresUrl(value: string): URL;
function assertNeonDirectUrl(value: string): URL;

interface DatabaseRuntime {
  readonly db: DrizzleDatabase;
  close(): Promise<void>;
}

type RuntimeDatabaseFactory = (databaseUrl: string) => DatabaseRuntime;
type DirectConnectionFactory = (databaseDirectUrl: string) => Promise<MigrationConnection>;

interface MigrationRequest {
  readonly connectionFactory: DirectConnectionFactory;
  readonly databaseDirectUrl: string;
  readonly migrationsFolder: string;
}

function createDatabaseRuntime(
  environment: AppEnvironment,
  factory?: RuntimeDatabaseFactory,
): DatabaseRuntime;
function runMigrations(request: MigrationRequest): Promise<void>;
```

`parsePostgresUrl` requires an absolute parsed URL with scheme `postgres:` or `postgresql:`, a
nonempty normalized hostname, a non-root database pathname, username, and password. URI parsing and
decoding preserve valid percent-encoded credentials, explicit ports, and bracketed IPv6 hosts.
`assertNeonDirectUrl` additionally inspects the normalized hostname and rejects the actual Neon
`-pooler.` marker; it does not infer pooling from `sslmode`, other query values, or arbitrary text in
credentials/path. The typed error is handled only at the executable boundary; libraries never
terminate the process.

Both `api` and `worker` execute validation, including the direct assertion, during startup. The
runtime provider uses `DATABASE_URL`; Drizzle Kit and `runMigrations` use only
`DATABASE_DIRECT_URL`. The injected connection factory lets PostgreSQL 17 Testcontainers supply a
direct test URL without changing `process.env`.

The local Compose file has no top-level `version`, uses PostgreSQL 17, contains no credential
literal, loads credentials through variable interpolation from a gitignored `.env`, and publishes
only to `127.0.0.1`. The tracked `.env.example` may contain unmistakably dummy local values so
`docker compose config` is runnable, but never a production/Neon URL or secret. The Drizzle config
selects dialect `postgresql`, the schema entry point, migrations output, and the direct credential;
checked-in migrations are produced by `drizzle-kit generate` and applied by
`drizzle-kit migrate` or the injectable migration adapter.

This section implements R3 and R5.

## D6. Health API and contract generation

The API composition root sets global prefix `api`, Nest URI versioning with default `1`, and a
controller path of `health`. The process binds to `127.0.0.1` by default. No controller includes
`api/v1` in its decorator.

```ts
declare const HealthStatusSchema: ZodType<HealthStatus>;
type HealthStatus = {
  readonly status: 'ok' | 'error';
  readonly database: 'up' | 'down';
};

interface DatabaseHealthProbe {
  check(): Promise<void>;
}

interface OpenApiSnapshotAdapter {
  create(application: INestApplication): OpenAPIObject;
  assertMatchesCheckedIn(document: OpenAPIObject): Promise<void>;
}
```

`contracts` owns the Zod 4 schema and JSON shape. The Nest adapter uses pinned
`nestjs-zod` 5.5.0 to derive DTO/OpenAPI metadata from it; there is no handwritten duplicate DTO.
The document returned by `SwaggerModule.createDocument` passes through `cleanupOpenApiDoc` before
snapshot comparison or Swagger setup. The database indicator uses the injected Drizzle provider
to execute a minimal query. Success maps to HTTP 200; a database exception maps to Terminus HTTP
503 with the sanitized shared error shape. TypeORM and its health indicator are absent.

`api:openapi-check` creates the application in a deterministic documentation profile, derives and
normalizes OpenAPI, compares it with the committed snapshot, and exits nonzero on any delta. It is
invoked in the main gate.

This section implements R6.

## D7. i18n and empty web shell

`packages/i18n` contains a real English locale file, catalog key types, and a framework-neutral
i18next factory. Flat dotted keys are allowed; the design does not require nesting. It has no React
peer dependency, export, or browser side effect.

```ts
type SupportedLocale = 'en' | 'tr' | 'ar';
type Phase0CatalogLocale = 'en';
type TextDirection = 'ltr' | 'rtl';

interface I18nCoreOptions {
  readonly requestedLocale: SupportedLocale;
  readonly fallbackLocale: Phase0CatalogLocale;
  readonly resources: Readonly<Record<string, unknown>>;
}

function createI18nCore(options: I18nCoreOptions): i18n;
function formatNumber(value: number, locale: SupportedLocale): string;
function formatDate(value: Date, locale: SupportedLocale): string;
function formatMoney(minorUnits: bigint, currency: string, locale: SupportedLocale): string;
function applyDocumentLocale(locale: string, direction: TextDirection): void;
```

The React provider/binding and `applyDocumentLocale` implementation live under `apps/web`. Locale
fallback, interpolation/plural rules, and `Intl`-backed number/date/bigint-money formatters remain
in the shared package and have explicit fixtures. The empty shell contains only structural markup
and catalog-backed English text. A test-only forced-RTL profile drives `dir="rtl"` without
pretending that the complete Arabic catalog exists. D-23's Phase-0 exception defers full Turkish
and Arabic catalogs to Phase 2.

The root flat ESLint configuration installs `eslint-plugin-i18next` and enables
`i18next/no-literal-string` with `framework: 'react'` and the officially supported
`mode: 'jsx-only'` for user-facing web source. Only structural attributes such as `className`,
`data-testid`, `id`, `key`, `role`, `lang`, and `dir` are excluded through the plugin's
`jsx-attributes.exclude` option; `aria-label`, `placeholder`, `title`, and `alt` remain checked.
Tests, catalog data, and config have narrow file-level exclusions. Controlled fixtures prove both a
visible JSX text literal and user-visible attribute literals fail lint. RTL geometry, horizontal
overflow, and overlap are measured only by Playwright in a real browser; jsdom tests cover semantic
content and `lang`/`dir` attributes only.

This section implements R7.

## D8. Test design

| Risk | Test level and owner | Assertion |
|---|---|---|
| tool/version/default drift | workspace-contract Vitest tests in `testing` | exact project set, targets, package pins, no Jest artifacts |
| illegal dependencies | flat ESLint plus policy fixture tests | every forbidden matrix edge, `db -> domain`, and backend React import fails; `db -> contracts` passes |
| hardcoded shell strings | ESLint policy fixture | visible JSX text plus `aria-label`/`placeholder` literals fail under `jsx-only`; catalog-backed text/attributes and narrow structural exclusions pass |
| URL/config classification | `db` Vitest with targeted fast-check | absolute URL, accepted schemes, required host/path/user/password, encoded credentials, IPv6/ports, and false-positive query strings; Neon `-pooler.` direct host rejected; typed errors, no exit |
| Compose safety | contract test plus `docker compose config --quiet` | PostgreSQL 17, loopback bind, no top-level version or Compose credential literal, required variables, dummy-only tracked example |
| schema and migrations | PostgreSQL 17 Testcontainers | every D4 table/column/FK/CHECK/index; no cascade-delete FK; NULLS NOT DISTINCT cases; run twice is idempotent with exactly the deterministic seed workspace; no feature seeds or raw-body/lifecycle columns |
| Drizzle runtime | `db` integration tests | runtime URL reaches provider; direct URL reaches migrations; factories injectable |
| health endpoint | `api:e2e` with Testcontainers and forced outage | exact route; 200/up and 503/down; loopback bootstrap |
| OpenAPI drift | `api:openapi-check` | deterministic generated snapshot equals tracked file |
| i18n semantics | framework-neutral Vitest | English fallback, interpolation/pluralization, and number/date/bigint-money formatting fixtures |
| web semantics | web Vitest + RTL + MSW | file catalog text and LTR/forced-RTL attributes; HTTP seams use MSW if introduced |
| web geometry | invoked Playwright `web:e2e` | no overflow/overlap in LTR and RTL at committed desktop/mobile viewports |
| CI/security contracts | workflow parsing tests plus Gitleaks | triggers/tiers/immutable actions, exact setup runtime/install inputs, one frozen install, least-privilege permissions, real scan commands, no primary secret in any future backup workflow |
| Neon PowerShell parser | Windows PowerShell-compatible self-test | libpq mapping, optional parameters, redaction, malformed/pooler rejection, setup copy/existence guard, both runbook callers dot-source the installed parser |

There is no repo-wide coverage threshold. No test asserts arbitrary millisecond/minute completion
bounds. HTTP integration tests use the application adapter or D-15 shared wrapper if an outbound
call is actually introduced; Phase 0 introduces no healthchecks.io HTTP call or direct global-fetch
exception.

This section implements R8 and contributes to R12.

## D9. CI, secrets, and monitoring

### Branch tiers

The `dev` workflow runs frozen install, lint with zero warnings, explicit typecheck, unit tests,
integration tests (including Testcontainers), and the pinned Gitleaks history/tree scan. The `main`
workflow runs the dev tier plus `api:e2e`, `web:e2e`, `api:openapi-check`,
`db:migration-check`, and `api`/`worker`/`web` production builds. Publishing, signing, packaging,
and release targets belong to later phases. Workflow contracts parse the checked-in YAML and assert
the commands really name existing Nx targets. No invented timing SLO is part of either tier.

All reusable action references are immutable full SHAs with tag comments. For this baseline,
`pnpm/setup@v2` is the pnpm-11-compatible successor and receives `runtime: node@24` plus
`install: false`; each job then performs exactly one explicit `pnpm install --frozen-lockfile`.
`pnpm/action-setup@v6` and a second package installation are explicitly excluded. The check/build
workflows declare `permissions: contents: read` at top or job scope; Phase 0 has no write-permission
exception. Gitleaks is pinned to 8.30.1 and executes both `gitleaks git` and `gitleaks dir`. GitHub
secret scanning and push protection are owner-controlled repository settings and require recorded
external evidence in addition to CI.

### Secret inventory and scope

| GitHub Secret name | Phase-0 purpose | Allowed consumer |
|---|---|---|
| `TELEGRAM_BOT_TOKEN` | future bot provisioning | server/notification jobs only; no browser bundle |
| `HEALTHCHECKS_IO_PRIMARY_KEY` | primary-cycle check | primary/server deployment only |
| `HEALTHCHECKS_IO_BACKUP_KEY` | backup-workflow check | backup workflow only |
| `ANDROID_KEYSTORE_BASE64` | future signed Android build placeholder | Phase-3 protected signing job only |
| `ANDROID_KEYSTORE_PASSWORD` | future keystore password | Phase-3 protected signing job only |
| `ANDROID_KEY_ALIAS` | future signing alias | Phase-3 protected signing job only |
| `ANDROID_KEY_PASSWORD` | future key password | Phase-3 protected signing job only |

Only names and scopes are tracked; values are never copied into a file or evidence log. A public
backup workflow must never reference, receive, or print `HEALTHCHECKS_IO_PRIMARY_KEY`. Browser code
receives none of these values. `.env`, Compose interpolation, logs, OpenAPI, test reports, and docs
are scanned for leakage.

### Healthchecks.io boundary

Phase 0 records two distinct owner-provisioned checks: primary cycle and backup workflow (including
eventual valid no-op). Their owner-selected healthchecks.io labels are not Windows Scheduled Task
names. The spec does not prescribe grace or cadence values. Because no cycle exists in Phase 0,
there is no ping client, placeholder scheduled workflow, or direct outbound `fetch`. Phase 1 wires
successful terminal paths through the D-15 shared HTTP wrapper: primary only after its complete
success conditions, backup after successful completion including valid no-op.

O-20 remains open. This design does not add an evidence-health check, quarantine action, retry cap,
or any escape from D-12 primary-ping suppression. Any such change requires the owner decision and
the explicit ledger amendments tracked by PT-005.

This section implements R9 and R10.

## D10. Steering, commands, and shared Neon URL parser

Steering files are a short index and usage guide, not a copied plan. They link to `AGENTS.md`,
`docs/CONTEXT.md`, `docs/canonical-plan.md`, `docs/decision-ledger.md`, and
`docs/work/board.md`, and direct an agent to read the canonical files. No arbitrary “first 50 lines”
snapshot or duplicated decision prose is generated.

Once targets exist, the `AGENTS.md` Commands placeholder is replaced with the verified install,
Nx lint/typecheck/test/build/e2e, Drizzle generation/migration-check, Docker Compose, health, and
secret-scan commands. The section remains operational, while policy stays in canonical documents.

PT-003's board instruction is absorbed into the implementation plan through one committed
`scripts/windows/parse-neon-url.ps1` parser.

PowerShell signature: `ConvertFrom-NeonDatabaseUrl -DatabaseUrl <string> -> PSCustomObject`.

The returned object always exposes required `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`,
`PGPASSWORD`, and `PGSSLMODE`; it exposes `PGCHANNELBINDING`, `PGCONNECT_TIMEOUT`, `PGAPPNAME`, and
`PGOPTIONS` only when their corresponding query parameters are supplied. It validates PostgreSQL
schemes and Neon direct-host expectations, percent-decodes through URI parsing, and never prints the
input or password.

The parser must preserve the runbook's TLS behaviour verbatim, because T29 deletes the only
executable copies of it and PT-004 is a Done story that depends on it. It defaults an omitted
`sslmode` to `require`; rejects empty parameter values, unsupported query parameters (fail closed,
never silently dropped), `sslmode=disable|allow|prefer` as weaker than `require`, and every
unrecognized `sslmode`; and normalizes `require`, `verify-ca`, and `verify-full` alike to
`PGSSLMODE=verify-full`, so hostname verification is mandatory. The runbook setup copies the
committed
`scripts/windows/parse-neon-url.ps1` from the checked-out repository to
`C:\ops\ai-price-tracker\parse-neon-url.ps1`, then fails unless that destination is a file. The saved
backup script and the runbook section 5 audit snippet both dot-source that installed copy; neither
retains a private parsing implementation. Static runbook contract tests verify the copy, existence
assertion, and both dot-source callers, while the owner verifies the installed path on the Windows
host. Parser tests remain compatible with the Windows PowerShell environment specified by the
runbook.

This section implements R11.

## D11. Error handling, phase gate, and non-goals

### Error handling

| Failure | Required result |
|---|---|
| invalid/missing environment | typed `EnvironmentValidationError`; field issues sanitized; executable boundary exits nonzero |
| relative/incomplete PostgreSQL URL, wrong scheme, or Neon pooler in direct URL | startup, migration, and CI fixture fail before opening a connection |
| local PostgreSQL unavailable | startup/migration fails clearly; health endpoint returns sanitized 503 if API is already serving |
| migration drift or second-run failure | `db:migration-check`/Testcontainers exits nonzero; no automatic destructive repair |
| OpenAPI delta | generated comparison exits nonzero and tells the implementer to inspect/commit intentional contract change |
| English catalog key missing | type/test failure; no fallback user literal in component source |
| illegal import or user literal | flat ESLint exits nonzero, including controlled negative fixtures |
| Gitleaks finding | CI exits nonzero; output is redacted before records are published |
| external provisioning unavailable | owner-controlled evidence remains incomplete; silence does not become an approval or invented configuration |

### Phase-0 gate

Completion requires all local targets and both CI tiers described in D8/D9, exact 200/503 health
tests, real-browser LTR/RTL evidence, schema/constraint and migration-idempotence evidence, Gitleaks
evidence, and owner-controlled evidence for the two checks, secret-name/scope inventory, GitHub
secret scanning/push protection, current board, and D-19's informed plus independent changes
reviews reconciled from fresh sessions. Documentation already live before PT-002 is not recreated.

### Explicit deferrals

- Phase 1 owns adapters, `run-due-checks`, claim/renew/finalize logic, runtime health pings, backup
  workflow execution/evidence, accepted-observation transitions, alert delivery, and its concurrency
  gate.
- Phase 2 owns full TR/AR catalogs, authentication behavior, add/discovery workflow, dashboards,
  AI routing/calls, notifications, PWA behavior, and realtime invalidation.
- Phase 3 owns native projects and actual Android signing consumption.
- Publishing, signing, packaging, and release-target automation is absent from the Phase-0 main
  gate and arrives only with its owning delivery phase.
- Phase 4 owns category discovery, seller/source-offer identity, analytics, and new change kinds.
- O-20 is an unresolved owner decision; no default is implied.

This section implements R12.

## D12. Two-way traceability

| Requirement | Design coverage | Implementation tasks | Verification owner |
|---|---|---|---|
| R1 | D1, D3 | T01–T05 | T05, T30 |
| R2 | D2 | T06–T07 | T07, T30 |
| R3 | D5 | T08–T10 | T08–T10, T30 |
| R4 | D4 | T11–T16 | T14–T16, T30 |
| R5 | D5 | T10, T15–T16 | T16, T30 |
| R6 | D6 | T17–T19 | T19, T30 |
| R7 | D7 | T20–T23 | T21–T23, T30 |
| R8 | D8 | T05, T14, T19, T22–T23 | T30 |
| R9 | D3, D9 | T24–T26 | T24–T26, T30 |
| R10 | D9 | T27–T28 | T27–T28, T30 |
| R11 | D10 | T29, T31–T32 | T29, T31–T32 |
| R12 | D8, D11 | T30, T33 | T30, T33 |

Conversely, every D1–D11 section names its governing requirement, and every implementation task
below cites both a requirement and a design section.
