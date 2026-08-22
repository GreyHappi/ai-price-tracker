# Prompt — Planner / Spec Season (Kiro IDE or any planner session)

> Use when a story is above the spec threshold (more than a few hours, or spans layers — D-21).
> Fill the `<...>` placeholders, paste into a **fresh** planner session.

---

You are the **planner** for story `<PT-### — title>` in the AI Price Tracker repo.

**Read first (in order):** `docs/CONTEXT.md` (invariants — non-negotiable),
`docs/canonical-plan.md` (relevant sections), `docs/decision-ledger.md` entries `<D-xx list>`,
and the story description below.

**Story:** <one paragraph: user value, motivation, constraints>

**Produce three files under `.kiro/specs/PT-###-<slug>/`:**

1. `requirements.md` — EARS acceptance criteria:
   - `WHEN <trigger>, THE SYSTEM SHALL <behavior within bound>`
   - `IF <error condition>, THE SYSTEM SHALL <handling>`
   - Cover the unhappy paths (timeouts, empty/garbage input, dedupe, i18n/RTL where UI-facing).
2. `design.md` — interfaces (TypeScript signatures), data delta (tables/columns touched),
   error handling, and a **Test plan** section (which layers, which fixtures, which properties).
   No code bodies.
3. `tasks.md` — atomic tasks, each with:
   - single-file (or narrow) focus, referencing `design.md` sections,
   - a **verify-by** command (`pnpm nx test <project> -- <filter>`),
   - checkbox format.

**Rules:**
- Do not relitigate locked decisions; if a decision blocks the design, name the D-xx and stop.
- Prefer the smallest design that satisfies the ACs (YAGNI list in canonical plan §11 is binding).
- Every user-facing string in the design goes through i18n catalogs (EN/TR/AR), never hardcoded.
- Output only the three files, no commentary.
