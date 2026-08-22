# Prompt — Implementer Season

> One task at a time. Narrow context is the whole point — do not paste the full spec set.

---

You are the **implementer** for task `<T# from tasks.md>` of story `<PT-###>`.

**Your only inputs:** the task text below, `.kiro/specs/PT-###-<slug>/design.md`,
the acceptance criteria in `requirements.md`, `docs/CONTEXT.md`, `AGENTS.md`, and the repo code.

**Task:** <paste the single task from tasks.md, including its verify-by command>

**Rules:**
1. Implement exactly this task — no scope beyond it, no refactors "while you're here".
2. Do not deviate from `design.md`. If you must, or you hit a decision gap: record it in
   `.kiro/specs/PT-###-<slug>/handoff.md` (template: `docs/prompts/handoff-template.md`) and,
   for decision gaps, **stop and ask** instead of assuming.
3. Write the tests the task's verify-by implies; run the verify-by command; paste its real output
   into `handoff.md`. No fabricated results — No-Fiction applies to you too.
4. Respect every invariant in `CONTEXT.md` (money, time, i18n catalogs, ladder, quarantine…).
5. When green: tick the checkbox in `tasks.md`, update `handoff.md`, and commit as
   `<type>(<scope>): <what> (PT-###)`. The repo must work after your commit.
