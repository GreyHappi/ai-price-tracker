# PT-002 — Phase 0 Foundation Tasks

## Execution contract

Execute tasks in order unless a task is explicitly owner-controlled. Each task is narrow, cites
its governing requirement/design section, and includes an assertive verification command. Run
commands from the repository root in PowerShell. A command that exits nonzero leaves the task
unchecked. Do not stage or commit partial work; the repository's normal Conventional Commit flow
applies only after the owner accepts a working task boundary.

Before running any verification block, use this required PowerShell 7 preamble so an early
native-command failure cannot be hidden by a later successful line:

```powershell
$ErrorActionPreference = 'Stop'
$PSNativeCommandUseErrorActionPreference = $true
```

Do not add a source adapter, cycle/scheduler, healthcheck HTTP client, placeholder backup schedule,
product UI, complete TR/AR catalogs, or an O-20 resolution while executing this list.

## Workspace and boundaries

- [ ] **T01 — Pin the root runtime and package manager.** Create the root pnpm workspace manifest,
  require Node 24.x, set `packageManager` to exactly `pnpm@11.23.0`, commit the generated
  `pnpm-lock.yaml`, and make frozen installation the repository default. Do not use Corepack's
  floating “latest”. **Refs:** R1.1; D3.

  **Verify:**

  ```powershell
  pnpm install --frozen-lockfile
  $manifest = Get-Content package.json -Raw | ConvertFrom-Json
  if ($manifest.packageManager -ne 'pnpm@11.23.0') { throw 'packageManager is not pinned' }
  if ((node -p "process.versions.node.split('.')[0]").Trim() -ne '24') { throw 'Node 24 is required' }
  ```

- [ ] **T02 — Initialize Nx with parity-pinned plugins.** Initialize Nx 23.1.1 and add only the
  plugins needed by this design (`@nx/js`, `@nx/node`, `@nx/nest`, `@nx/react`, `@nx/vite`,
  `@nx/vitest`, `@nx/playwright`, `@nx/eslint`), all exactly 23.1.1. Establish the root flat ESLint
  and TypeScript bases without generating projects yet. **Refs:** R1.1, R1.5; D3.

  **Verify:**

  ```powershell
  $nxVersion = (pnpm exec nx --version).Trim()
  if ($nxVersion -ne '23.1.1') { throw "Expected Nx 23.1.1, got $nxVersion" }
  pnpm install --frozen-lockfile
  ```

- [ ] **T03 — Generate the three applications with explicit runner choices.** Generate fixed Nx
  projects `api` (Nest), `worker` (Node), and `web` (React + Vite + CSS, no router). Explicitly
  disable generated Jest/default e2e projects; select Vitest for web and no generated runner for
  the two backend apps, then add explicit `lint`, `typecheck`, and `build` targets to each
  application. Capture the resolved noninteractive Nx 23 generator options in task evidence. Do not
  create a bot, native, mobile, or landing app. **Refs:** R1.2–R1.4; D1, D3.

  **Verify:**

  ```powershell
  $projects = @(pnpm nx show projects)
  foreach ($name in @('api','worker','web')) { if ($name -notin $projects) { throw "Missing $name" } }
  $jestFiles = @(Get-ChildItem -LiteralPath . -Recurse -File | Where-Object {
    $_.FullName -notmatch '[\\/](\.git|node_modules)[\\/]' -and
    $_.Name -match '(^jest\.config\.|\.jest\.config\.|\.spec\.jest\.)'
  })
  if ($jestFiles.Count -ne 0) { throw "Unexpected Jest artifact: $($jestFiles -join ', ')" }
  pnpm nx run-many -t lint,typecheck,build --projects=api,worker,web
  ```

- [ ] **T04 — Generate the eight library projects.** Generate fixed projects `contracts`, `domain`,
  `db`, `scraping`, `ai`, `i18n`, `notifications`, and `testing` under `packages/`, with buildable
  TypeScript entry points and explicit lint/typecheck targets. Do not generate speculative package
  implementations. **Refs:** R1.2–R1.4; D1, D3.

  **Verify:**

  ```powershell
  $expected = @('ai','api','contracts','db','domain','i18n','notifications','scraping','testing','web','worker') | Sort-Object
  $actual = @(pnpm nx show projects) | Sort-Object
  if (Compare-Object $expected $actual) { throw "Nx project set differs: $($actual -join ', ')" }
  pnpm nx run-many -t lint,typecheck,build --projects=contracts,domain,db,scraping,ai,i18n,notifications,testing
  ```

- [ ] **T05 — Install and wire the Phase-0 test harness.** Configure Vitest targets for frontend,
  backend, and libraries; install PostgreSQL Testcontainers, targeted `fast-check`, React Testing
  Library, MSW, and Playwright dependencies without Jest; and add a workspace contract test that
  asserts the exact project set, version pins, and absence of Jest artifacts. For every required
  target, assert the stable target name and runnable configuration reported by `nx show project`;
  inferred and manually declared targets are both valid under that same proof. Do not add a
  repository-wide coverage threshold. **Refs:** R1.3–R1.4, R8.1–R8.5;
  D3, D8.

  **Verify:**

  ```powershell
  pnpm nx test testing -- workspace.contract.spec.ts
  pnpm nx run-many -t test
  ```

- [ ] **T06 — Encode the complete module-boundary matrix.** Tag all eleven projects and implement
  D2 in the flat ESLint config, including app isolation, `domain` with no workspace dependencies,
  ADR-0004's `db -> contracts`-only edge, production/test-support separation, framework-neutral
  `i18n`, and backend React/web import bans. **Refs:** R2.1–R2.4; D2.

  **Verify:**

  ```powershell
  pnpm nx run-many -t lint --all -- --max-warnings=0
  ```

- [ ] **T07 — Prove boundary enforcement with negative fixtures.** Add isolated ESLint-API contract
  fixtures for every disallowed class in D2, including a `domain` workspace import, app-to-app
  import, `db -> domain`, production import of `testing`, `i18n` React import, and backend React/web
  import. Include a passing `db -> contracts` control. The test must require each forbidden fixture
  to fail for the expected rule. **Refs:** R2.2, R2.5; D2, D8.

  **Verify:**

  ```powershell
  pnpm nx test testing -- boundaries.contract.spec.ts
  ```

## Environment, database, and schema

- [ ] **T08 — Implement typed environment validation and direct-URL classification.** Add the D5
  interfaces, Zod loader, and absolute PostgreSQL URL validation requiring an accepted scheme plus
  nonempty hostname, database path, username, and password. Preserve encoded-credential,
  explicit-port, and IPv6 cases; reject Neon `-pooler.` only for the parsed direct hostname. Execute
  the assertion from both application bootstraps and a CI contract test. Libraries throw
  `EnvironmentValidationError`; only executable boundaries choose exit codes. Use targeted
  fast-check cases for URL syntax/classification. **Refs:** R3.1–R3.4; D5, D8.

  **Verify:**

  ```powershell
  pnpm nx test db -- environment.spec.ts
  pnpm nx test api -- environment-startup.spec.ts
  pnpm nx test worker -- environment-startup.spec.ts
  ```

- [ ] **T09 — Add the safe PostgreSQL 17 local Compose profile.** Create the Compose service,
  preserve and assert the existing `.env` ignore rule, provide safe local-only example values, bind
  PostgreSQL only to `127.0.0.1`, and omit both a top-level Compose `version` and tracked password
  literals in the Compose file. Mark every tracked example credential unmistakably dummy/local and
  reject production/Neon-looking secrets. Add a contract test for image major, bind, interpolation,
  and forbidden keys. Add `testing:compose-smoke` to start the `postgres` service with
  `.env.example`, wait for readiness, execute `SELECT 1`, and tear down volumes from a `finally`
  path even when an assertion fails. **Refs:** R3.5–R3.6; D3, D5, D8.

  **Verify:**

  ```powershell
  docker compose --env-file .env.example config --quiet
  pnpm nx test testing -- compose.contract.spec.ts
  pnpm nx run testing:compose-smoke
  ```

- [ ] **T10 — Add the pooled Drizzle runtime provider.** Implement `createDatabaseRuntime` in
  `db` with the validated `DATABASE_URL`, an injectable factory, and clean pool shutdown; wire the
  same provider into `api` and `worker`. Do not put schema declarations in either app and do not use
  the direct URL for runtime queries. **Refs:** R5.3; D5.

  **Verify:**

  ```powershell
  pnpm nx test db -- runtime.integration.spec.ts
  pnpm nx test api -- database-provider.spec.ts
  pnpm nx test worker -- database-provider.spec.ts
  ```

- [ ] **T11 — Define shared schema-value contracts.** In `contracts`, define Zod contracts for
  bigint-as-string money JSON, positive/non-negative integers, non-padding uppercase three-letter
  currency, tri-state stock, lease UUIDs, exact `en|tr|ar` locales, exact `primary|backup` lanes,
  exact `pending|sending|sent|failed` outbox statuses, and the reserved `probe`/`ai` literals
  required by D4. For canonically unnamed health/kind/status/error/channel values, define only a
  bounded trimmed nonempty seam.
  Export one contract fixture that database tests can compare with Drizzle constraints; do not add
  Phase-4 change kinds or other guessed enums. **Refs:** R4.9–R4.11; D4.

  **Verify:**

  ```powershell
  pnpm nx test contracts -- schema-values.spec.ts
  pnpm nx typecheck contracts
  ```

- [ ] **T12 — Declare workspace, target, entry, and group-fence tables.** Add `workspaces`, `users`,
  `tracking_targets`, `source_entries`, and `fetch_leases` with exactly the D4 binding columns/FKs,
  root-only workspace placement, target soft archive, 1:N target-to-entry ownership,
  `(target_id,url,selector) NULLS NOT DISTINCT`, no URL-only unique constraint, and composite
  group-fence primary key. Use `NO ACTION`/`RESTRICT`, never cascade delete, on workspace/target/
  entry tracking FKs. **Refs:** R4.1–R4.5, R4.9–R4.10; D4.

  **Verify:**

  ```powershell
  pnpm nx test db -- core-targets.contract.spec.ts
  pnpm nx typecheck db
  ```

- [ ] **T13 — Declare observation, event, and outbox tables.** Add `observations`, `change_events`,
  and `notification_deliveries` with the exact D4 fields, exact four-field hash contract,
  observation-chain NULLS-NOT-DISTINCT uniqueness, unique event observation, unique delivery
  dedupe, and pending/sending/sent/failed lease-and-attempt checks. Record D4's stable dedupe inputs
  without implementing enqueue or inventing the future rule model. All history/outbox FKs use
  `NO ACTION`/`RESTRICT`, never cascade delete. Do not add a snapshot lifecycle column or raw
  response field. **Refs:** R4.3, R4.6–R4.7, R4.9–R4.10; D4.

  **Verify:**

  ```powershell
  pnpm nx test db -- history-outbox.contract.spec.ts
  pnpm nx typecheck db
  ```

- [ ] **T14 — Declare run, AI, settings, and discovery seams.** Add `scrape_runs`, `ai_calls`,
  `app_settings`, `discovery_requests`, and `discovery_candidates` with only the D4 binding fields.
  Enforce the exact lane set, reserved `probe` literal, bounded nonempty run status/kind and nullable
  error-class seams, duration, and the nullable-entry probe check; do not guess terminal-status
  enums. Preserve integer cost/tokens/latency, root-only workspace placement, and non-cascading FKs.
  Do not implement AI routing or the discovery workflow. Add a completeness test for all thirteen
  canonical names and absence of an invented join table/raw-body field. **Refs:** R4.1–R4.3,
  R4.8–R4.11; D4.

  **Verify:**

  ```powershell
  pnpm nx test db -- run-support.contract.spec.ts schema-v1.contract.spec.ts
  pnpm nx typecheck db
  ```

- [ ] **T15 — Configure Drizzle Kit and generate the initial migration.** Add the PostgreSQL
  `drizzle.config.ts`, bind generation/migration to the validated direct URL, generate the first
  checked-in migration with current `drizzle-kit generate`, and add its conflict-safe deterministic
  insert of exactly one fixed MVP workspace UUID. Do not seed a user, chat, settings, or other
  feature row. Ensure generated tracking/history FKs remain `NO ACTION`/`RESTRICT`, not cascade.
  Add `db:migration-check` that compares disposable generation output without modifying the
  migration directory. **Refs:** R4.3, R4.12, R5.1; D3–D5.

  **Verify:**

  ```powershell
  pnpm nx run db:migration-check
  ```

- [ ] **T16 — Implement injectable, idempotent migrations against PostgreSQL 17.** Implement
  `runMigrations` with the D5 injected direct connection factory. A Testcontainers suite must apply
  the initial migration twice, introspect all D4 columns/FKs/CHECK/unique indexes, exercise both
  NULLS-NOT-DISTINCT conflicts, prove tracking/history FKs do not cascade, and prove invalid
  money/currency/outbox/run rows are rejected. It must assert the same fixed workspace ID and an
  exact workspace row count of one after both runs, with zero invented user/settings feature seeds.
  Because the first migration is high-risk under D-19, a fresh independent/adversarial review must
  be recorded before T17 begins.
  **Refs:** R4.3, R4.12, R5.2, R5.4, R8.2; D4, D5, D8.

  **Verify:**

  ```powershell
  pnpm nx test db -- migrations.integration.spec.ts
  ```

## API and web shell

- [ ] **T17 — Add the flat health Zod contract and derived Nest DTOs.** Define the exact
  `{ status: 'ok', database: 'up' }` and `{ status: 'error', database: 'down' }` Zod 4 response
  schemas plus their union in `contracts`, serialize their JSON shape deterministically, pin
  `nestjs-zod` 5.5.0, and derive the Nest/OpenAPI DTO metadata from those schemas. Wire its current
  validation/serialization integration and do not create a parallel handwritten interface.
  **Refs:** R6.1, R6.5; D3, D6.

  **Verify:**

  ```powershell
  pnpm nx test contracts -- health.spec.ts
  pnpm nx test api -- health-dto.spec.ts
  pnpm nx typecheck api
  ```

- [ ] **T18 — Implement the loopback Drizzle health endpoint.** Configure global prefix `api`, URI
  versioning default `1`, controller path only `health`, and default host `127.0.0.1`. Implement the
  Terminus custom indicator over the injected Drizzle probe, but do not expose Terminus's default
  `@HealthCheck()` envelope. Map the internal result to T17's exact flat 200/503 DTOs. Do not
  install TypeORM or hardcode `api/v1` in the controller. **Refs:** R6.2–R6.4; D6.

  **Verify:**

  ```powershell
  pnpm nx test api -- health-controller.spec.ts bootstrap.spec.ts
  pnpm nx typecheck api
  ```

- [ ] **T19 — Add API e2e, health smoke, and OpenAPI snapshot targets.** Configure `api:e2e` with a
  real PostgreSQL 17 Testcontainer and assert exact 200/up plus forced 503/down behavior at
  `/api/v1/health`. Add non-hanging `api:health-smoke` that owns a PostgreSQL 17 Testcontainer,
  starts on loopback, asserts the exact flat HTTP 200 body, and always closes both application and
  container. Generate OpenAPI with `SwaggerModule.createDocument`, apply
  `cleanupOpenApiDoc`, and add the deterministic checked-in snapshot plus `api:openapi-check`.
  **Refs:** R1.3, R6.4–R6.5, R8.2, R12.2; D3, D6, D8.

  **Verify:**

  ```powershell
  pnpm nx run api:e2e
  pnpm nx run api:health-smoke
  pnpm nx run api:openapi-check
  ```

- [ ] **T20 — Build the framework-neutral English catalog package.** Add a real English locale
  file and i18next initialization in `i18n`; dotted flat keys are permitted. Export catalog types,
  English fallback/interpolation/plural behavior, `Intl`-backed number/date/bigint-money formatters,
  and `createI18nCore`, but no React binding, React dependency, component, or browser side effect.
  Add explicit fixtures for fallback, pluralization, and formatting. **Refs:** R7.1, R7.3, R7.5;
  D2, D7.

  **Verify:**

  ```powershell
  pnpm nx test i18n -- i18n-core.spec.ts
  pnpm nx lint i18n -- --max-warnings=0
  ```

- [ ] **T21 — Implement the catalog-backed empty web shell and literal-string lint gate.** Keep the
  React binding and document `lang`/`dir` synchronization in `web`. Render only the Phase-0 shell
  from the English catalog. Install/configure `eslint-plugin-i18next` in flat ESLint and prove
  `i18next/no-literal-string` uses `framework: 'react'`, `mode: 'jsx-only'`, and only D7's narrow
  structural `jsx-attributes.exclude` list. Negative fixtures must independently reject visible JSX
  text, `aria-label`, and `placeholder` literals; passing fixtures must prove catalog-backed values
  and the named structural exclusions. **Refs:** R7.2–R7.4; D7, D8.

  **Verify:**

  ```powershell
  pnpm nx test testing -- hardcoded-user-string.contract.spec.ts
  pnpm nx lint web -- --max-warnings=0
  ```

- [ ] **T22 — Add semantic web tests and the MSW harness.** Configure React Testing Library and MSW
  under the web/testing setup without adding a production HTTP call. Assert catalog-backed content,
  normal LTR attributes, and the forced-RTL test profile's `lang`/`dir`. Do not assert layout,
  overflow, or element geometry in jsdom. **Refs:** R7.5, R8.3; D7, D8.

  **Verify:**

  ```powershell
  pnpm nx test web -- phase-zero-shell.spec.tsx
  pnpm nx test testing -- msw-harness.spec.ts
  ```

- [ ] **T23 — Add and invoke real-browser LTR/RTL Playwright coverage.** Configure the actual
  `web:e2e` target and test the empty shell at committed desktop and mobile viewports in LTR and
  forced RTL. Assert the catalog text is visible, logical alignment flips, and no element overlaps
  or causes horizontal overflow. Install the pinned Chromium browser before invoking the target;
  package installation alone is not browser provisioning. **Refs:** R1.3, R7.3, R7.5, R8.3,
  R12.3; D3, D7, D8.

  **Verify:**

  ```powershell
  pnpm exec playwright install chromium
  pnpm nx run web:e2e -- --grep "Phase 0 shell"
  ```

## CI, security, and external provisioning

- [ ] **T24 — Implement the `dev` CI tier.** Create `.github/workflows/dev-checks.yml`; on pushes
  to `dev` and manual dispatch, use
  immutable-SHA-pinned `actions/checkout@v7` plus `pnpm/setup@v2`. Configure the setup action with
  `runtime: node@24` and `install: false`, then execute exactly one explicit
  `pnpm install --frozen-lockfile` before workspace lint/typecheck/test, including database
  integration. Declare top/job `permissions: contents: read`. Do not use `pnpm/action-setup@v6`, a
  floating action tag, write permission, second install, or fabricated timing bound. The workflow
  contract test must assert triggers, full-SHA pins, exact setup inputs, exactly one frozen install,
  least privilege, and exact Nx commands. **Refs:** R9.1, R9.3, R9.6; D3, D9.

  **Verify:**

  ```powershell
  pnpm nx test testing -- dev-workflow.contract.spec.ts
  ```

- [ ] **T25 — Implement the `main` phase-gate CI tier.** Create
  `.github/workflows/main-gate.yml`; on pushes/merges to `main` and manual dispatch, run the
  complete dev tier plus `testing:compose-smoke`, `api:e2e`, `web:e2e`, `api:openapi-check`,
  `db:migration-check`, and the `api`/`worker`/`web` production
  builds. Provision ephemeral PostgreSQL 17 runtime/direct URLs for targets that validate them and
  install Chromium with Playwright's Linux dependencies before `web:e2e`. Reuse T24's exact
  `pnpm/setup` inputs, single frozen install, immutable pins, and `permissions: contents: read`;
  publishing/signing/packaging/release targets are absent. The workflow must invoke only the real
  targets created earlier. **Refs:** R9.2–R9.3, R9.6; D3, D9.

  **Verify:**

  ```powershell
  pnpm nx test testing -- main-workflow.contract.spec.ts
  ```

- [ ] **T26 — Add real Gitleaks history and working-tree gates.** Pin Gitleaks 8.30.1 in the
  executable CI/local target and run current `gitleaks git` plus `gitleaks dir` with redacted
  output. Add a recognized dummy-secret fixture proving the scanner fails, then allow only that
  explicit test fixture. Do not use deprecated `detect`, grep-only scanning, or a nonasserting
  “report” step. Amend both `dev-checks.yml` and `main-gate.yml` to invoke the executable scan, and
  extend both workflow contract tests to fail when that invocation is absent. **Refs:** R9.1–R9.5;
  D3, D8, D9.

  **Verify:**

  ```powershell
  pnpm nx run testing:secret-scan
  pnpm nx test testing -- secret-scan.contract.spec.ts
  ```

- [ ] **T27 — Record and enforce secret names and scopes without values.** Add a repository secret
  inventory containing only the seven D9 names/scopes. Extend environment/workflow contract tests
  so no browser bundle receives a server secret and any backup workflow is rejected if it references
  `HEALTHCHECKS_IO_PRIMARY_KEY`; do not create a placeholder backup schedule or ping implementation.
  Ensure `.env`, Compose, docs, and build output remain covered by secret scanning. **Refs:** R3.6,
  R10.2–R10.4; D9.

  **Verify:**

  ```powershell
  pnpm nx test testing -- secret-scope.contract.spec.ts
  pnpm nx run testing:secret-scan
  ```

- [ ] **T28 — Provision and verify owner-controlled external settings.** This task is external and
  must not place values or full ping URLs in the repository. Ensure two distinct healthchecks.io
  checks exist with the roles “primary cycle” and “backup workflow including valid no-op”: reuse
  and record a correctly configured existing check or create a missing one. Retain owner-selected
  names/cadence/grace without renaming checks after `AIPT-primary-up`/`AIPT-neon-backup`; if those
  values are unavailable, leave owner evidence incomplete instead of guessing. Add all seven
  secret names to GitHub, keep the primary key outside the public backup workflow, and enable
  GitHub secret scanning plus push protection.
  O-20 remains open and no third check is provisioned by this task. **Refs:** R9.5, R10.1–R10.5;
  D9, D11.

  **Owner-controlled external verify — GitHub secret names:**

  ```powershell
  $required = @('TELEGRAM_BOT_TOKEN','HEALTHCHECKS_IO_PRIMARY_KEY','HEALTHCHECKS_IO_BACKUP_KEY','ANDROID_KEYSTORE_BASE64','ANDROID_KEYSTORE_PASSWORD','ANDROID_KEY_ALIAS','ANDROID_KEY_PASSWORD')
  $actual = @((gh secret list --json name | ConvertFrom-Json).name)
  $missing = @($required | Where-Object { $_ -notin $actual })
  if ($missing.Count -ne 0) { throw "Missing GitHub Secrets: $($missing -join ', ')" }
  ```

  **Owner-controlled external verify — repository security settings:**

  ```powershell
  $repoSlug = (gh repo view --json nameWithOwner --jq '.nameWithOwner').Trim()
  $repoState = gh api "repos/$repoSlug" | ConvertFrom-Json
  if ($repoState.security_and_analysis.secret_scanning.status -ne 'enabled') { throw 'Secret scanning is not enabled' }
  if ($repoState.security_and_analysis.secret_scanning_push_protection.status -ne 'enabled') { throw 'Push protection is not enabled' }
  ```

  **Owner-controlled external verify — healthchecks.io:** In the healthchecks.io project UI, compare
  the two redacted check identifiers and their configured roles. Record evidence that the IDs are
  different, the primary check is not the backup check, and neither role was inferred from a
  Windows Scheduled Task name. Do not export ping URLs, grace, or cadence into the repository.

## Operations and completion

- [ ] **T29 — Centralize Neon URL parsing and update both runbook callers.** Add
  `scripts/windows/parse-neon-url.ps1` with `ConvertFrom-NeonDatabaseUrl` and redaction-safe tests.
  Update the setup section in `docs/runbooks/windows-server.md` to copy that committed file from the
  checkout into `C:\ops\ai-price-tracker\parse-neon-url.ps1` and fail unless the destination exists.
  Update the saved `backup-neon.ps1` block and section 5 audit snippet to dot-source that installed
  parser; remove their duplicate parse blocks. Preserve the existing libpq variable mapping, with
  channel binding and the other mapped optional parameters emitted only when supplied. Preserve the
  runbook's TLS behaviour exactly — default an omitted `sslmode` to `require`, throw on empty values,
  unsupported query parameters, `disable|allow|prefer`, and unrecognized modes, and normalize
  `require`/`verify-ca`/`verify-full` alike to `verify-full` — because deleting the duplicated blocks
  otherwise regresses PT-004's verified-TLS guarantee. The self-test must cover each of those
  rejection and normalization cases, and must statically assert the
  setup copy, existence guard, and both dot-source callers. This closes PT-003's board instruction
  without changing unrelated runbook behavior. Expose the self-test through
  `testing:neon-parser-check`, selecting `powershell.exe` on Windows and `pwsh` elsewhere while
  retaining Windows PowerShell compatibility. Amend `main-gate.yml` to invoke the new target and
  extend its workflow contract test only after the target exists. **Refs:** R9.2, R11.2–R11.3;
  D3, D8–D10.

  **Verify:**

  ```powershell
  pnpm nx run testing:neon-parser-check
  ```

  **Owner-controlled external verify — installed Windows host:**

  ```powershell
  if (-not (Test-Path -LiteralPath 'C:\ops\ai-price-tracker\parse-neon-url.ps1' -PathType Leaf)) { throw 'Installed Neon URL parser is missing' }
  ```

- [ ] **T30 — Run the complete automated Phase-0 gate.** Run every local gate after the executable
  implementation exists. The health smoke must start/stop its own loopback test application; no
  hanging, watch-mode, or nonasserting command is accepted. **Refs:** R12.1–R12.3; D8, D11.

  **Verify:**

  ```powershell
  pnpm nx run-many -t lint,typecheck,test
  pnpm nx run-many -t e2e --projects=api,web
  pnpm nx run api:openapi-check
  pnpm nx run db:migration-check
  pnpm nx run api:health-smoke
  pnpm nx run testing:secret-scan
  pnpm nx run testing:compose-smoke
  pnpm nx run testing:neon-parser-check
  ```

- [ ] **T31 — Seed Kiro steering as a source index.** Create concise steering files that link to and
  prescribe the read order for `AGENTS.md`, `CONTEXT.md`, canonical plan, decision ledger, active
  spec, and current board. Do not copy a fixed leading-line slice or maintain a second decision
  summary. Add a contract test that asserts the required sources and rejects large copied blocks,
  plus a non-mutating `testing:markdown-check` target that parses the Phase-0-touched Markdown and
  fails on malformed structure, broken internal links, or unmatched fences. **Refs:** R11.1; D10.

  **Verify:**

  ```powershell
  pnpm nx test testing -- steering.contract.spec.ts
  pnpm nx run testing:markdown-check
  ```

- [ ] **T32 — Fill the `AGENTS.md` Commands section from executable targets.** Replace the Phase-0
  placeholder only after T30 is green. Document verified pnpm install, Nx lint/typecheck/test/build,
  API/web e2e, Drizzle generate/migration check, Docker Compose, loopback health smoke, and secret
  scan commands. Keep decisions as links rather than copied prose and add a contract test that each
  named Nx target exists. **Refs:** R11.4; D10.

  **Verify:**

  ```powershell
  pnpm nx test testing -- agents-commands.contract.spec.ts
  pnpm nx show project api | Out-Null
  if ($LASTEXITCODE -ne 0) { throw 'Documented API project is not executable' }
  ```

- [ ] **T33 — Reconcile evidence and close the Phase-0 board gate.** Re-run the complete automated
  gate after documentation changes, check Markdown and diff whitespace, record the green dev/main
  CI runs plus T28's redacted external evidence, run and reconcile D-19's informed and independent
  changes reviews in fresh sessions, and only then update PT-002/current Phase-0 board state. Leave
  O-20 open and do not rewrite already-live canonical docs. **Refs:** R11.5,
  R12.1–R12.5; D11.

  **Verify:**

  ```powershell
  pnpm nx run testing:markdown-check
  git diff --check
  pnpm nx run-many -t lint,typecheck,test
  pnpm nx run-many -t e2e --projects=api,web
  pnpm nx run api:openapi-check
  pnpm nx run db:migration-check
  pnpm nx run api:health-smoke
  pnpm nx run testing:secret-scan
  pnpm nx run testing:compose-smoke
  pnpm nx run testing:neon-parser-check
  ```

  **Owner-controlled external verify — CI:**

  ```powershell
  gh workflow run dev-checks.yml --ref dev
  $devRun = (gh run list --workflow dev-checks.yml --branch dev --limit 1 --json databaseId | ConvertFrom-Json)[0].databaseId
  gh run watch $devRun --exit-status
  gh workflow run main-gate.yml --ref main
  $mainRun = (gh run list --workflow main-gate.yml --branch main --limit 1 --json databaseId | ConvertFrom-Json)[0].databaseId
  gh run watch $mainRun --exit-status
  ```

## Completion rule

PT-002 is complete only when T01–T33 are checked with retained evidence and the Phase-0 roadmap gate
is green. An unavailable owner-controlled service leaves the corresponding task open; lack of
response is not approval and does not authorize a configuration guess. Phase 1 may then consume the
schema and provisioning seams, but it still owns all runtime cycle/ping/evidence behavior.
