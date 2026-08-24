# Prompt — Planner / Spec Season (Kiro IDE or any planner session)

> Use when a story is above the spec threshold (more than a few hours, or spans layers — D-21).
> Fill the `<...>` placeholders, paste into a **fresh** planner session.
>
> **Structure baseline (owner decision 2026-08-24, D-21):** Kiro's documented spec format is the
> reference. Everything below is additive to it, never a replacement — where this prompt is silent,
> follow Kiro.

---

You are the **planner** for story `<PT-### — title>` in the AI Price Tracker repo.

**Read first (in order):** `docs/CONTEXT.md` (invariants — non-negotiable),
`docs/canonical-plan.md` (relevant sections), `docs/decision-ledger.md` entries `<D-xx list>`,
and the story description below.

**Story:** <one paragraph: user value, motivation, constraints>

**Produce three files under `.kiro/specs/PT-###-<slug>/`:**

1. `requirements.md` — an introduction stating scope and non-scope, then numbered requirements.
   Each requirement carries a **user story** (`As a <role>, I want <capability>, so that <benefit>`)
   and numbered **acceptance criteria** in EARS:
   - `WHEN <trigger>, THE SYSTEM SHALL <observable behavior>`
   - `IF <error condition>, THEN THE SYSTEM SHALL <handling>`
   - `WHILE <state>` and `WHERE <optional feature is present>` for state- and option-scoped criteria.

   Number criteria `<requirement>.<criterion>` (1.1, 1.2, …) so tasks can reference them.
   Cover the unhappy paths (timeouts, empty/garbage input, dedupe, i18n/RTL where UI-facing).
   State a bound only where something actually measures it; never invent a wall-clock number.
2. `design.md` — Kiro's design sections apply: components and their responsibilities, data flow and
   interactions, data models, API contracts and interfaces, error handling, and non-functional
   considerations. Add a **Test plan** section (which layers, which fixtures, which properties) —
   in this repo it lives inside `design.md`, never as a separate file (D-21).
3. `tasks.md` — atomic tasks in checkbox format, each with:
   - single-file (or narrow) focus, citing the `design.md` section it implements,
   - the acceptance-criteria numbers it satisfies (`Refs: R2.1, R2.3`),
   - a **verify-by** command that fails loudly (`pnpm nx test <project> -- <filter>`).

**How concrete should `design.md` be?**

Be exact where the design is **binding**; do not paste implementations that merely illustrate.

- Binding — write it out: table/column/constraint lists, type and function signatures, wire
  contracts, config keys, pinned versions, and the exact rule a guard enforces.
- Not binding — leave it to the implementer: runnable module bodies, framework wiring, import
  lists, and complete config files. The planner runs no compiler, and unverified code inside a
  design file carries authority it has not earned, then gets pasted.

Diagrams are welcome wherever they carry the mechanism better than prose.

**Rules:**

- **Transcribe canon; never re-derive it.** A spec touching the data model reproduces the
  `canonical-plan.md` §4 tables and columns as written. Renaming, adding, or dropping a table is
  forbidden. If canon looks wrong, name the D-xx and stop.
- Do not relitigate locked decisions; if a decision blocks the design, name the D-xx and stop.
- Prefer the smallest design that satisfies the ACs (YAGNI list in canonical plan §11 is binding).
- Every user-facing string in the design goes through i18n catalogs (EN/TR/AR), never hardcoded.
- Where canon is silent (a type width, a grace value), pick one consistently across all three files
  or raise it as an owner decision — never assert an unsourced number.
- Output only the three files, no commentary.
