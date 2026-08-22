# CLAUDE.md

@AGENTS.md

Claude-specific deltas only — the imported contract above is canonical:

- Independent reviews — risk-triggered in-phase AND at phase gates — run in **fresh sessions**.
  For the independent review, use
  `docs/prompts/git-changes-review.md` and do **not** open `.kiro/specs/*/design.md`,
  `handoff.md`, or any chat history — only the inputs the prompt allows.
- Project-specific skills are a deferred backlog item (ledger D-25); until they exist, follow
  AGENTS.md + `docs/prompts/` directly.
- Scratch/temp output goes to the session scratchpad, never into the repo.
