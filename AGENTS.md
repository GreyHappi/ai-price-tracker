# AGENTS.md — Repo Contract for AI Agents

> Canonical instructions for **every** AI tool working in this repo (Kiro IDE, Claude Code,
> Codex/GPT, Cursor, …). Tool-specific deltas live in that tool's own file (e.g. `CLAUDE.md`,
> which imports this one) and must never duplicate it.

## What this project is

**AI Price Tracker** — tracks prices and stock of products on Turkish sites (notebooks; used cars
on arabam.com, currently unsupported-blocked per O-14; and new-car OEM price lists for five brands:
Kia Sportage, Toyota RAV4, VW Tayron, Škoda Kodiaq, BMW X3). It detects semantic changes and
notifies via Telegram; the dashboard is a web SPA + PWA, later Tauri desktop/Android/iOS. Seed
targets live in `docs/product/watchlist.md`. Solo project, $0 infrastructure target (AI API terms
must be verified under O-03), built through AI seasons with a heavy emphasis on process artifacts.

## Read first, in this order

1. [docs/CONTEXT.md](docs/CONTEXT.md) — invariants (hard 200-line cap). **Non-negotiable.**
2. [docs/canonical-plan.md](docs/canonical-plan.md) — architecture & process. All decided.
3. [docs/decision-ledger.md](docs/decision-ledger.md) — every decision with rationale, rejected
   alternatives, and reopen triggers (D-xx / N-xx / O-xx).
4. The active spec under `.kiro/specs/PT-###-slug/` when implementing or reviewing.

## Ground rules

- **Knowledge flows through files, never chat memory.** If it isn't written, it doesn't exist.
- **Decisions are locked in the ledger.** Do not relitigate or silently deviate; to challenge a
  decision, name the D-xx and its reopen signal.
- **AI proposes, human approves.** Never auto-track a product; never auto-apply parser repairs;
  AI-extracted prices are quarantined and never fire alerts unconfirmed.
- **No CAPTCHA/login/anti-bot bypass code — ever.** Such sources are marked `unsupported`.
- **Secrets never in code or docs.** GitHub Secrets / gitignored `.env` only.

## Conventions

- Code EN · docs EN · user-facing strings only via i18n catalogs (**EN/TR/AR** — UI, bot replies,
  and notifications; UI must be RTL-safe).
- Money = bigint minor units + ISO-4217 currency, never float. Time = UTC `timestamptz`.
- Commits: Conventional Commits + task id — `feat(scraper): vatan adapter (PT-014)`.
  Every commit leaves the repo working.
- Branches: work on `dev`; `main` only via phase-gate merges. **No PRs, no feature branches.**
- Nx module boundaries are law; `packages/domain` depends on nothing.
- Specs only for work above a few hours or spanning layers; small work = board entry + commits.

## Season protocol (planner / implementer / reviewer)

- **Planner** produces the spec triplet (`requirements.md` EARS / `design.md` / `tasks.md`) in
  `.kiro/specs/PT-###-slug/` — prompt: [docs/prompts/planner-spec.md](docs/prompts/planner-spec.md).
- **Implementer** receives ONE task + `design.md` + `CONTEXT.md`; deviations go to `handoff.md`,
  never silently into code — prompt: [docs/prompts/implementer.md](docs/prompts/implementer.md).
- **Reviews are risk-based:** high-risk stories get an immediate independent/adversarial review;
  medium-risk work gets one at story/small-epic completion; low-risk work uses automated gates.
  Phase gates still run both fresh-session prompts: informed review
  ([docs/prompts/informed-review.md](docs/prompts/informed-review.md)) + independent changes review
  ([docs/prompts/git-changes-review.md](docs/prompts/git-changes-review.md)). Exact commit ranges
  are used in-phase; `main..dev` at the phase gate. No PR is required. An implementer never
  supplies its own independent review in the same session.

## Commands

Phase 0 will fill this section (pnpm/nx/docker/test commands). Until then there is nothing to run.

## Repo map

`apps/` — api (NestJS + grammY bot module), worker (scheduler → `run-due-checks`), web
(React SPA/PWA); `native` appears at the Tauri phase.
`packages/` — contracts, domain, db, scraping, ai, i18n, notifications, testing.
`docs/` — canonical plan, ledger, ADRs (`decisions/`), prompts, roadmap, runbooks, work/board,
and `records/` (append-only history: `records/seasons/` season logs + appendices, spike results,
process policy).
`.kiro/` — steering + specs (generated via Kiro IDE).
