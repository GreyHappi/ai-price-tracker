# Review — PT-002 Phase 0 Foundation (spec triplet)

**Date:** 2026-08-24 · **Subject:** `requirements.md` / `design.md` / `tasks.md` **as first
generated** · **Reviewer:** Claude Opus 5, multi-lens adversarial review · **Status of that first
draft:** rejected.

> **Read [§9](#9-round-2--the-rewrite-2026-08-24) first.** The draft reviewed in §2–§8 no longer
> exists: it was regenerated in response to this review, and the replacement is what this
> repository carries. Sections 2–8 describe the rejected draft and are kept because the ledger of
> what was wrong is why the replacement is trustworthy — not as a description of the committed
> files. Section 9 records the rewrite, its independent verification, and every item still open.

## Verdict

**Format: pass. Content: fail.**

The three files are in the right place, carry the right names, and the requirements file is written
in recognisable EARS shape with a traceability skeleton. That is the whole of the good news.

The substance is largely invented. The planner read `canonical-plan.md` §4 and then wrote a
different data model: canonical table names were renamed, five §4 tables were dropped, a
many-to-many join table that canon does not have was added, and most of the columns that carry the
locked invariants (`snapshot_hash`, `lane`, `dedupe_key`, `lease_owner`, `workspace_id`,
`archived_at`, `extraction_method`) are simply absent. Several of the technical artifacts pasted
into `design.md` do not compile or name APIs that do not exist.

**Counts (deduplicated): 30 P1 · 32 P2 · 11 P3.** Four candidate findings were examined and
rejected; they are listed in §7 so the record is honest.

## 1. Method

Six independent review lenses ran in fresh contexts over the three spec files, each with a
different mandate, followed by an adversarial verification pass whose default position was that
each finding is wrong, plus a completeness critic hunting the seams between lenses:

| Lens | Mandate |
|---|---|
| data model | table/column conformance against canonical-plan §4 and CONTEXT § Data |
| invariants | conformance with CONTEXT (all sections) and D-01…D-26, N-15, O-07/O-18/O-19 |
| external | every API, CLI flag, package, lint rule, image tag and action version verified upstream |
| spec craft | EARS quality, traceability in both directions, task atomicity, verify-by commands |
| scope | roadmap Phase 0 scope + gate, canonical-plan §11 YAGNI, forward blockers for Phase 1 |
| critic | contradictions between the three files; what every other lens missed |

Findings that survived verification are recorded below with the authoritative source they
contradict. Nothing here is a stylistic preference.

Severity: **P1** = produces wrong or non-building code, a wrong schema, or violates a locked
invariant. **P2** = real gap or inaccuracy that costs time but is recoverable. **P3** = minor.

## 2. P1 — schema v1 does not match canon

The Phase-0 schema is load-bearing: `roadmap.md` scopes "Drizzle schema v1 + first migration
(canonical plan §4 tables)" into this story, and CONTEXT twice names Phase 0 as the place a
mechanism "is pinned". Every item below either breaks at run time or forces a second migration
before Phase 1 can start.

| id | Finding | Consequence |
|---|---|---|
| S-01 | Chain uniqueness declared on `prev_observation_id` **alone**; canon pins `UNIQUE NULLS NOT DISTINCT (entry_id, prev_observation_id)` | `NULLS NOT DISTINCT` treats NULLs as equal, so exactly **one row table-wide** may have a NULL predecessor — only the first source entry in the system could ever get a baseline observation. Fails on the second entry, after Phase 0 is declared green |
| S-02 | Test plan states the constraint "allows multiple nulls" | Backwards. The test written from this line certifies the bug as correct behaviour, or is "fixed" by deleting `nullsNotDistinct()` |
| S-03 | Canonical names replaced: `tracking_targets`→`targets`, `change_events`→`events`, `notification_deliveries`→`deliveries` | Every later query, ADR and gate criterion refers to the canonical names |
| S-04 | Five §4 tables absent: `workspaces`, `users`, `ai_calls`, `app_settings`, `discovery_requests`/`candidates` | D-10's workspace rule cannot be implemented; CONTEXT § AI ("model IDs live in DB config, never in code") has no `app_settings` to live in; D-23's per-user `locale` has no `users` row |
| S-05 | Invented many-to-many `targets_source_entries`; `source_entries.target_id` dropped | D-13 locks a two-level 1:N model. N:M makes "identity scoped per target" inexpressible and admits one entry belonging to two targets — auto-matching by another name, which D-13 forbids |
| S-06 | Source-entry identity never pinned — the spec states only that `UNIQUE(url)` must be absent | CONTEXT: "*the Phase-0 schema pins it*". Without `(target_id, url, selector)` NULLS NOT DISTINCT the `/add` flow can insert the same entry twice, with no DB-level defence |
| S-07 | `source_entries` missing `target_id`, `site`, `health`, `last_checked_at`, `next_check_at`, `lease_owner`, `lease_until` | The entire Phase-1 due/claim path has no storage |
| S-08 | `observations` missing `snapshot_hash`, `extraction_method`, `confidence`, `observed_at` | D-12's write-only-on-semantic-change rule and CONTEXT's AI-quarantine rule are unimplementable |
| S-09 | `in_stock` typed NOT NULL boolean | Canon requires tri-state (`null` = source publishes no availability) and makes null a hash participant. All five OEM price-list sources publish no stock field; an adapter must invent `false`, which reads as "out of stock" and can fire a bogus alert |
| S-10 | Currency split into `payable_price_currency` + `list_price_currency` | Canon defines a single `currency` per observation and the hash tuple is written over exactly one |
| S-11 | Invented `observations.lifecycle_state` with `pending / accepted / held / discardable` | Those four are the **on-disk snapshot envelope** states from D-12, not a DB column. The set also omits `suspicious`, which the sanity guards actually need, so the real hold mechanism stays unmodelled |
| S-12 | `tracking_targets` loses `workspace_id`, `kind`, `target_price_minor`, `archived_at`; gains a dangling `user_id varchar` with no FK and no `users` table | Silently relitigates D-10 with a rejected alternative. Phase 2's "below target price" rule has no column |
| S-13 | `change_events.observation_id` carries no UNIQUE constraint | The one-canonical-event-per-accepted-observation replay key does not exist; a retry mints a second event |
| S-14 | `notification_deliveries` loses `recipient_id`, `dedupe_key` (unique), `status`, `attempts`, `lease_until`; `sent_at` becomes NOT NULL `delivered_at` | The whole D-05 outbox-lite machine is gone. Phase 1's gate criteria "exactly one alert" and "a replay produces no new delivery row" cannot be met on this schema |
| S-15 | `fetch_leases` uses a surrogate uuid PK instead of the composite `(adapter_key, canonical_url)`, and documents `claimed_by` as a **"worker ID"** | CONTEXT is explicit: "*The token is never a stable worker/lane id.*" A stable id defeats fencing entirely — a delayed worker from a reclaimed lease re-presents the same id and its effect transaction validates |
| S-16 | `scrape_runs` missing `adapter_key`, **`lane`**, `status`, `duration`, `error_class` | `lane` is the field that makes the dual-lane runtime auditable. The runbook §5 "did the backup lane cover the gap?" query has no column to filter on; D-02's auto-pause counters keyed by (entry, lane, error class) are unimplementable |
| S-17 | No `archived_at` on any table | CONTEXT: "Soft-archive with `archived_at`; never hard-delete tracking history." The only removal path left is DELETE |
| S-18 | The only CHECK constraint in the schema is `scrape_runs.entry_id_or_probe` | The named `price > 0` guard and all enum CHECKs are absent, so the "final defense layer" behind the sanity guards does not exist |

## 3. P1 — locked decisions contradicted

| id | Finding | Source it contradicts |
|---|---|---|
| C-01 | `packages/i18n/src/index.ts` re-exports `react-i18next` from the public entry, and `initI18n` never registers the React binding | CONTEXT § i18n + D-23 + D-04 all require the package to be framework-neutral: "*Backend renderers do not depend on React.*" It is the entry file, so no import path avoids it. The re-exported `useTranslation` also has no instance to read — it would not work even for web |
| C-02 | The unpooled-migration guard tests for `?pool` / `&pool` | Neon marks pooled endpoints with a **`-pooler.` host segment**; the guard can never fire on a real pooled URL. The repo's own runbook already implements the correct check (`-match '-pooler\.'`). CONTEXT and N-15 additionally require the assertion at **startup and in CI**; the spec places it only inside `runMigrations()`, and neither workflow asserts anything |
| C-03 | Health controller uses `TypeOrmHealthIndicator` | The locked ORM is Drizzle (D-01, ADR-0003). The indicator requires `@nestjs/typeorm` and a TypeORM `DataSource`; the module cannot be instantiated. An implementer following `design.md` literally either installs a second ORM or stalls |
| C-04 | OpenAPI-derived-from-Zod and its CI snapshot test are absent from all three files; `packages/contracts` appears only as a directory to create | D-14 and canonical-plan §12 ("Continuous gates: … OpenAPI snapshot unchanged"). Phase 0 builds both the API and the CI, and ships a real endpoint. Retrofitting later means rewriting controllers written without nestjs-zod |
| C-05 | The two healthchecks.io checks are named `AIPT-primary-up` and `AIPT-neon-backup` with 2 h / 25 h grace | Those are **Windows Task Scheduler task names** (runbook §77, §324), and `AIPT-neon-backup` is the 03:30 daily local `pg_dump` job — not the Actions scrape lane D-02's second check must watch. The grace values appear nowhere in canon, and 25 h on a 60-minute cron keeps a disabled schedule green for over a day, which is precisely the failure (N-04 public-repo auto-disable) the check exists to surface. The Phase-1 gate verifies those same names via `schtasks /Query /XML`, so the collision also makes that evidence ambiguous |
| C-06 | `HealthcheckConfig` makes both ping keys required, so `pingBackup()` cannot be called without the primary key | Hands the **public-repo** backup workflow the credential that can mask a primary outage. Destroys the independence property the two-check design exists for (D-02) |
| C-07 | Nothing constrains the API listen address; NestJS defaults to all interfaces | CONTEXT § API and D-10: "*the API binds to loopback behind Serve.*" On the primary machine this means an unauthenticated API reachable on the LAN before the AuthGuard exists |
| C-08 | None of the locked test tooling is installed. Vitest, Playwright, MSW, fast-check and coverage appear **zero times** across all three files; no task creates an e2e project or adds Testcontainers as a dependency; the API/worker are scaffolded "with Nx generators" without pinning a runner | D-17 locks Vitest for backend as well as frontend, and the Nx Nest/Node generators do not offer Vitest — the API and worker land on Jest. Every verify-by command in `tasks.md` is unrunnable on the workspace this spec produces, and the gate command has no test runner behind it |
| C-09 | The healthcheck ping calls global `fetch` directly, with no timeout or AbortSignal | CONTEXT § API: "*All HTTP goes through the shared fetch wrapper*" (D-15). The try/catch handles errors but not hangs, so a stalled connection blocks the caller indefinitely — contradicting the spec's own Req 9.5. It also establishes the repo's first HTTP call site as an exception to the rule |
| C-10 | `/api/v1` is hardcoded into the controller path | D-14 locks the **mechanism** (NestJS URI versioning), not just the string. The versioning seam is never installed; adding it later either double-prefixes existing routes or requires touching every controller |
| C-11 | Nothing in the triplet creates a runtime database connection — no Drizzle client, no pool, no NestJS provider, no task producing one | Four acceptance criteria and two tasks depend on one existing |
| C-12 | `validateEnv()` calls `process.exit(1)` and never throws | Req 2.2/2.3 and Property 5 require a thrown ZodError. The two are mutually exclusive: `process.exit` inside a Vitest worker kills the run, so the prescribed test can never pass. A library-level `process.exit` also makes `packages/db` unusable by any caller wanting to handle config failure |

## 4. P2 — technical content that does not work as written

Verified against current upstream documentation.

- **T-01** The Drizzle schema block uses `primaryKey`, `unique` and `sql` without importing them
  (and `sql` comes from `drizzle-orm`, not `drizzle-orm/pg-core`) — four TS2304 errors on paste.
- **T-02** The self-referencing FK on `observations.prev_observation_id` lacks the explicit
  return-type annotation TypeScript needs, producing a circular-inference error.
- **T-03** `drizzle-kit generate:pg` no longer exists; the current CLI needs a `drizzle.config.ts`
  that no task creates.
- **T-04** `@typescript-eslint/no-hardcoded-strings` is not a real rule, and
  `--rule no-hardcoded-strings` is not valid ESLint CLI syntax (`--rule` takes a JSON object). The
  real options are third-party: `eslint-plugin-i18next`'s `i18next/no-literal-string`, or
  `react/jsx-no-literals`.
- **T-05** Nx boundary enforcement is driven by project `tags` + `depConstraints`, not by emptying
  a package's dependency list; `.eslintrc.json` is the legacy config format.
- **T-06** `pnpm/action-setup@v2` is deprecated and is used with no `version` input while no
  `packageManager` field is specified anywhere — the step errors before install.
- **T-07** CI pins Node 20, which is end-of-life.
- **T-08** `@Controller('api/v1/health')` plus the global prefix the Nx Nest generator writes
  yields `/api/api/v1/health`.
- **T-09** Local dev and integration tests pin PostgreSQL 16; the runbook pins `postgres:17` and
  requires matching Neon's major.
- **T-10** The compose `version:` key is obsolete and warns on every command.
- **T-11** `z.string().url()` is the deprecated Zod 3 form; the stack is pinned to Zod 4
  (`z.url()`).
- **T-12** Eleven verify-by commands and both workflows invoke a `typecheck` target that no task
  configures; Nx ships no default `typecheck` target.
- **T-13** The Nx project naming scheme is never decided, yet ~30 verify-by commands and both
  workflows hardcode bare project names. Nx derives the name from `package.json`, so a scoped
  choice breaks all of them.
- **T-14** All four table-extras callbacks return an object, deprecated in drizzle-orm 0.36 in
  favour of an array.
- **T-15** The env Zod schema validates the two database URLs with a bare generic URL check, which
  accepts `https://example.com` as a Postgres connection string. The repo already ships a stricter
  validator for the same variable in the runbook — and Phase 0 is the phase tasked with extracting
  it.
- **T-16** The committed `docker-compose.yml` hardcodes a password and publishes 5432 on every
  interface, in a repo that is public by locked decision and against the spec's own Req 10.3.

## 5. P2 — spec craft

- **Q-01** Fourteen acceptance criteria carry invented wall-clock bounds that measure the wrong
  thing or are unverifiable ("create directories within 5 seconds", "parse env within 100 ms",
  "create an `en` catalog directory within 1 second"). No verify-by measures any of them.
- **Q-02** Six ACs use pseudo-triggers ("WHEN the repository is initialized", "WHEN Kiro IDE loads
  the workspace") — not runtime events, and the actor performing the SHALL is a developer or a
  third-party IDE.
- **Q-03** Six of forty tasks have no runnable verify-by; eight more are commands that exit 0
  regardless of whether the work was done (`cat`, `docker compose ps`, a `curl` that prints a
  status code without asserting it); `pnpm nx graph` does not terminate at all.
- **Q-04** Four tasks verify by running spec files no task creates; neither `api-e2e` nor `web-e2e`
  is ever created, though both are targeted.
- **Q-05** Requirement 8 — the requirement the Phase-0 gate is written around — has no design
  section, no component, and no `dir` mechanism anywhere in `design.md`.
- **Q-06** Five acceptance criteria, four of them the unhappy paths the planner prompt explicitly
  demanded, have no task and no test: Docker not running, DB down → 503, missing EN catalog,
  secret committed, steering loaded.
- **Q-07** Property 2 (module boundaries) cannot detect the violation it guards — reading
  `packages/domain/package.json` says nothing about what the source files import.
- **Q-08** Property 7 asserts RTL layout facts (scrollbars, overlapping boxes) but prescribes a
  jsdom Vitest run, which performs no layout: every rect is zero and the assertions are vacuously
  true. The one test guarding the gate's RTL criterion certifies nothing.
- **Q-09** Req 4.10 states the wrong mechanism for migration idempotence and contradicts Req 2.6,
  which states the right one.
- **Q-10** `requirements.md` mandates `char(3)` for currency; `design.md` implements `varchar(3)`.
  Beyond the disagreement, `char(3)` space-pads on read, so a later equality comparison behaves
  differently — in exactly the "parsed currency differs from the entry's ⇒ suspicious" check.
- **Q-11** The workflow timeouts are double the acceptance bounds they are meant to enforce
  (300 s vs `timeout-minutes: 10`; 600 s vs 15), and nothing else measures the bounds.
- **Q-12** Req 10.4 names a single `HEALTHCHECKS_IO_KEY` while the design and tasks define two —
  one key cannot serve two independent dead-man checks. The roadmap's keystore secrets are absent
  from all three files.
- **Q-13** `design.md` ships full implementation bodies (schema, env module, two workflow YAMLs,
  controller, ping functions, compose file) against the planner prompt's explicit "No code
  bodies" — and the pasted code does not compile.
- **Q-14** The Module Boundaries section declares constraints for 3 of 8 packages. `contracts`,
  `ai`, `i18n`, `notifications` and `testing` have no declared allowed-dependency set, so Task 4
  cannot be implemented as written — and the one boundary CONTEXT explicitly demands (backend must
  not depend on React) is never expressed as a rule.
- **Q-15** Test-plan gaps: no i18n fallback/plural/number-format fixtures (D-23 requires explicit
  tests), no `apps/api` unit layer despite Task 18, no home for Property 2, no OpenAPI snapshot,
  and a "Visual Tests" section duplicating the E2E RTL entry with a contradicting runner.
- **Q-16** The glossary defines `fetch_leases` as "a row-level lock preventing concurrent scrapes".
  Canon defines an expiring, token-bearing fence row and explicitly rejects lock ordering as an
  implementation.
- **Q-17** Task 24 offers a free choice between two mutually exclusive RTL strategies, one of which
  CONTEXT pins (logical properties) and the other of which it does not; no task installs Tailwind,
  which the task presumes exists.
- **Q-18** Task 24's verify-by runs a spec file only Task 25 creates.

## 6. P2/P3 — scope

- **X-01** PT-003 — which the board explicitly directs to be *folded into the PT-002 spec* — appears
  nowhere. The runbook names the shared `parse-neon-url.ps1` extraction a Phase-0 action, and one of
  the two duplicated copies is the pooled-URL guard the TypeScript side already got wrong (C-02).
- **X-02** `AGENTS.md` states that Phase 0 fills its Commands section. No requirement, no design
  section and none of the 40 tasks does so.
- **X-03** No EN catalog **file** is ever created; three literals are hardcoded inside the body of
  `initI18n()`. The gate's "seeded EN catalog in `packages/i18n`" is therefore not delivered.
- **X-04** Seeding steering from "the first 50 lines of `docs/CONTEXT.md`" truncates mid-sentence
  and drops five of its six sections — including the no-CAPTCHA rule, the AI-quarantine rule, the
  `/api/v1` contract rule, and all of i18n and Process. A hand-copied partial also drifts.
- **X-05** `runMigrations()` is an empty stub with a zero-argument signature reading only
  `process.env`, which makes Task 16's Testcontainers test structurally unable to point it at the
  ephemeral container.
- **X-06** *Over-scope:* Phase 0 is made to ship a no-hardcoded-string lint rule, which the Phase-0
  gate parenthetical explicitly defers ("the rule binds from the first real user-facing string").
- **X-07** *Over-scope:* Req 9.3/9.4 and Tasks 29-30 build and test the healthcheck **pinging**
  path, but Phase 0 has no primary cycle and no backup workflow to ping from; the roadmap puts the
  pings in Phase 1's gate.
- **X-08** *Invented gate criterion:* a 120-second wall-clock ceiling that the roadmap gate does not
  ask for and that contradicts the spec's own Testcontainers budget.
- **X-09** Req 10.2 / Task 33 assert `.env` still has to be added to `.gitignore`; it has been there
  since the planning commit.

## 7. Claims examined and rejected

Recorded so the review is honest about its own false positives.

| Claim | Why rejected |
|---|---|
| The EN catalog's flat dotted keys (`'app.title'`) cannot resolve, since i18next's `keySeparator` is `.` | i18next's `ignoreJSONStructure` defaults to **true** — "if a key is not found as nested key, it will try to lookup as flat key". Nothing in the design overrides it, so `t('app.title')` resolves |
| The `dev` CI tier never runs integration tests | The spec puts them on the `test` target (`nx test db -- migrations.spec.ts`), which `run-many -t lint,typecheck,test` executes, on a runner that already provides Docker |
| Ten tasks cite `design.md` section names that do not exist | Inexact but recognisable; not a defect worth an implementer's time |
| The roadmap's "Docs live" bullet is uncovered | The docs already exist; there is nothing for a task to do |

## 8. Root cause and recommendation

One cause explains most of §2. The planner prompt tells the planner to *read*
`canonical-plan.md`, but never requires it to **transcribe** §4. A weaker model read the section,
formed its own idea of a sensible schema, and wrote that instead — while `requirements.md`'s own
glossary still defines "Schema v1" as "the first Drizzle migration containing canonical-plan §4
tables". The spec contradicts itself on its central deliverable.

Recommended, in order:

1. **Do not commit the triplet as generated.** The files are staged; unstage them.
2. **Harden [docs/prompts/planner-spec.md](../../../docs/prompts/planner-spec.md):** any spec that
   touches the data model must transcribe the canonical-plan §4 table and column list verbatim;
   renaming a table, adding one, or dropping one is forbidden, and a planner that believes canon is
   wrong must stop and name the D-xx rather than deviate. This single rule prevents the same class
   of failure recurring in Phase 1.
3. **Regenerate the spec** with the §4 block and the relevant CONTEXT invariants embedded directly
   in the prompt rather than referenced.
4. Fold **PT-003** in, per the board (X-01).

Nothing in this review reopens a locked decision. Where the spec and canon disagree, canon stands;
where canon is silent (for example `char(3)` vs `varchar(3)`, or healthcheck grace values), the
spec must either pick one consistently across its three files or raise it as an owner decision
rather than assert an unsourced number.

## 9. Round 2 — the rewrite (2026-08-24)

### What happened

The owner tasked GPT 5.6 Sol with synthesising this review against two other tool reviews and
regenerating the triplet. It reports that Opus 5 xhigh was not available in its spawn set, so it
used gpt-5.6-sol xhigh instead — recorded here because the instruction named a different model.
The three files were replaced (`requirements.md` 178 → 358 lines, `design.md` 546 → 517,
`tasks.md` 259 → 536) and re-staged.

### Independent verification of the rewrite

Four fresh lenses ran over the replacement — *did the findings actually get fixed*, *does the
rewrite assert new external facts that are false*, *did it introduce new canon violations or
over-scope*, *is it internally consistent and implementable* — each followed by an adversarial
pass whose default position was that the finding is wrong, then a gate critic. 38 candidate
findings; 15 were refuted, 23 survived, plus 4 from the critic.

**No P1 survives.** Result by finding class:

| Class | Round 1 | Round 2 outcome |
|---|---|---|
| Schema v1 vs canonical-plan §4 (S-01…S-18) | 18 P1 | all closed. Thirteen canonical tables under canonical names; `UNIQUE NULLS NOT DISTINCT (entry_id, prev_observation_id)` and `(target_id, url, selector)`; `fetch_leases` composite PK with an explicit fresh-UUID-never-worker-id token; tri-state `in_stock` in the exact four-field hash tuple; one `currency`; `archived_at`, `lane`, `dedupe_key`, `attempts`, `lease_until`, `snapshot_hash`, `extraction_method`; no join table; no invented lifecycle column or enums where canon is silent |
| Locked decisions (C-01…C-12) | 12 P1 | all closed. Framework-neutral `i18n`; hostname `-pooler.` guard asserted at both bootstraps and in CI; Drizzle health indicator, no TypeORM; contracts-first Zod → `cleanupOpenApiDoc` → checked-in snapshot; healthcheck names divorced from the `AIPT-*` Scheduled Tasks; backup lane receives only the backup key, with a test rejecting any reference to the primary one; loopback bind; test tooling actually installed; a real runtime DB provider; typed error instead of `process.exit`; URI versioning instead of a hardcoded path |
| External/technical (T-01…T-16) | 16 P2/P3 | closed, and every replacement pin was verified against upstream rather than accepted: Node 24 LTS, pnpm 11.23.0, Nx 23.1.1 across all `@nx/*`, `actions/checkout@v7.0.1`, `nestjs-zod` 5.5.0 with `cleanupOpenApiDoc` documented as required, Gitleaks 8.30.1 with `git`/`dir`, `eslint-plugin-i18next` `framework`/`mode: 'jsx-only'`/`jsx-attributes.exclude`, `@nx/vitest:configuration` and `@nx/playwright:configuration`. The highest-risk claim resolved in the spec's favour: `pnpm/setup` is a real repository distinct from `pnpm/action-setup`, and `action-setup`'s own README names it the successor required for pnpm 11 |
| Spec craft (Q-01…Q-18) | 18 P2/P3 | closed. Fabricated time bounds and pseudo-EARS triggers gone; every task carries an assertive PowerShell block that throws; RTL geometry moved to Playwright; requirements↔design↔task tables added |
| Scope (X-01…X-09) | 9 | closed. PT-003 folded in as T29; `AGENTS.md` Commands as T32; steering as a source index rather than a truncated copy; O-20 explicitly left open by R10.5, D9, T28 and T33 |

One round-1 suspicion was **wrong** and is corrected here: the `db → contracts` only edge is not
invented — ADR-0004 states it verbatim ("`packages/domain` depends on **nothing**; `packages/db`
only on `contracts`"). The spec quotes canon accurately. Five further over-scope claims raised in
round 2 (the lint rule, the security block, the contract-test layer, `db:migration-check`, the
steering index) were examined and rejected as canon-compatible.

### Verdict

**Accepted for commit.** The schema is canon-faithful, the external baseline is real, and the
tasks are executable. What remains is second-order and is listed below rather than hidden.

## 10. Open items carried into implementation

None blocks acceptance. They are recorded so an implementer meets them as known work, not as
surprises. The first six are worth resolving before or during the tasks that touch them.

| id | Item | Why it matters |
|---|---|---|
| IC-02 | 21 of 33 verify blocks call `pnpm exec vitest run <path>` from the repo root, but no task creates a root Vitest `projects` config. T22 proves the failure: one invocation mixes a jsdom `.tsx` spec with a node `.ts` spec, which a single Vitest project cannot satisfy | Cheapest fix in the list: [planner-spec.md](../../../docs/prompts/planner-spec.md) already prescribes `pnpm nx test <project> -- <filter>`, and `@nx/vitest` infers exactly that per-project target — the correct form needs no config at all |
| D10 parser | The shared `ConvertFrom-NeonDatabaseUrl` contract enumerates the libpq mapping but omits two behaviours the duplicated runbook blocks actually perform: rejecting sslmode weaker than `require` with mandatory escalation to `verify-full`, and rejecting unknown query parameters. It also marks `PGCHANNELBINDING` required where the runbook treats it as optional | T29 deletes those duplicated blocks. An implementer following D10 literally would silently undo PT-004 — a Done story — and downgrade the nightly `pg_dump` from verified TLS |
| NEW-01 | R10.2 orders `HEALTHCHECKS_IO_PRIMARY_KEY` into the GitHub Secrets of a repo that is public by D-24, though D9's own table names its only consumer as the home server, which reads a local `.env`. The roadmap asks for "bot token & keystore secrets" and nothing more | Puts the one credential R10.3 exists to isolate into the place R10.3 calls dangerous, for no consumer |
| NEW-03 | No task or gate ever *starts* the local Compose PostgreSQL — it is only ever parsed with `docker compose config --quiet` — yet `api:health-smoke` runs twice in the gate and needs a live database Testcontainers does not supply | A working Compose Postgres is itself a Phase-0 roadmap deliverable |
| IC-03 | No CI tier actually runs the secret scan: D9 says the `dev` workflow does, R9.1's dev list omits it, and T26 creates `testing:secret-scan` *after* both workflow tasks — so neither workflow could reference it | The Gitleaks gate exists as a local target only |
| NF-01 / IC-01 | D6 declares a flat `{status, database}` Zod response **and** mandates Terminus, whose `@HealthCheck()` envelope is `{status, info, error, details}`. Both cannot hold, and the checked-in OpenAPI snapshot would document a body the endpoint never returns | Decide at T17/T18: a custom controller returning the flat contract, or adopt the Terminus envelope as the contract |
| L5 / L10 | R10.1 and T28 ask the owner to *identify* two existing healthchecks.io checks; the roadmap Phase-0 bullet asks for them to be *created* | Round 1's C-05 correction overshot |
| L3 | Playwright is installed but no step installs the browser binaries, though `web:e2e` carries the gate's RTL evidence | First `web:e2e` run fails |
| L7 | `db:migration-check` runs in CI with no step provisioning `DATABASE_DIRECT_URL`, while the env loader is specified to throw when it is absent | — |
| L4 | `api:openapi-check` is scheduled on the `main` tier only, so contract drift is not a continuous gate | Canon-compatible (D-20 pins e2e/build to `main`); recorded as a judgement call |
| IC-04 | Three of twelve rows in D12's traceability table are inaccurate (R3→T10, R8→T14 instead of T16, R11 missing T33). T01–T33 themselves are complete with no gaps or duplicates | The table asserting traceability is not itself accurate |
| IC-08 | T33 hardcodes two workflow filenames (`dev-checks.yml`, `main-gate.yml`) that no earlier task, requirement or design section names | — |
| IC-12 | T03's Jest-artifact check scans only `apps/`, so a root-level `jest.config.ts` written by the Nx generators would pass undetected; it also shells out to `rg`, which no task installs | The check can pass with Jest present |
| IC-09 · IC-10 · IC-11 · IC-14 · L6 | Bookkeeping: D1 claims R12 coverage the tables omit; requirements/tasks say two health schemas where design declares one union; `testing:markdown-check` appears in no requirement or design section yet gates T33; the Neon parser self-test is required by D8 but invoked by no gate; the EN-only Phase-0 carve-out is attributed to a "D-23 exception" that does not exist — the carve-out lives in the roadmap gate parenthetical, and D-23 is owner-locked with no exception | Citation accuracy |

These are implementation-time corrections, not spec-blocking defects. Deviations taken while
resolving them go to `handoff.md` per D-21, never silently into code.
