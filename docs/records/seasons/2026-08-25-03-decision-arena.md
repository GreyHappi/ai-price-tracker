# Season Record — 2026-08-25 #03 — PT-002 pre-implementation decision arena

- **Type:** decision arena — two reviewers, three rounds, verdicts applied to the spec
- **Operator:** GreyHappi · **Assistants:** Claude Fable 5 low (orchestrator), Claude Fable xhigh
  (reviewer), GPT 5.6 Sol xhigh (reviewer), Claude Opus 5 xhigh (applied the verdicts)

## Goal

Settle the four pre-implementation items the PT-002 spec season left open, so implementation can
start at T01 without a decision still waiting at T05, T17/T18, or T28.

## Inputs

- [review.md](../../../.kiro/specs/PT-002-phase-0-foundation/review.md) — §10's open items and
  §11's second-opinion outcomes, which named all four questions
- The spec triplet:
  [requirements.md](../../../.kiro/specs/PT-002-phase-0-foundation/requirements.md) ·
  [design.md](../../../.kiro/specs/PT-002-phase-0-foundation/design.md) ·
  [tasks.md](../../../.kiro/specs/PT-002-phase-0-foundation/tasks.md)
- [Roadmap](../../roadmap.md) Phase 0 and its gate, [decision ledger](../../decision-ledger.md)
  (D-02, D-12, D-24), [CONTEXT](../../CONTEXT.md)
- Nx 23 migration documentation on nx.dev, checked live during round 2
- The concurrent pre-implementation reconciliation recorded in
  [2026-08-25 #02](./2026-08-25-02-phase0-preimplementation-reconciliation.md), which became the
  merge base

## Outputs

- [review.md §13](../../../.kiro/specs/PT-002-phase-0-foundation/review.md) — the arena, its four
  verdicts, and the concurrency ruling that governs how they were applied
- Spec edits: R1.3, R6.4, R10.1, R10.2 in `requirements.md`; D3, D6, D9 in `design.md`; T05, T18,
  T27, T28 in `tasks.md`
- Two corrected clauses in ledger note N-19, which now points at review.md §13
- The Phase-0 healthchecks bullet in [roadmap.md](../../roadmap.md), reworded to achieved state
- [Board](../../work/board.md) PT-002 Ready cell updated
- This record

## Decisions touched

No ledger decision reopened or amended. D-02's two-independent-checks property is preserved by
both the healthchecks verdict and the secret trim; D-24's "secrets in GH Secrets" permits but does
not require the trimmed key, which is why the trim needed an owner-level call rather than a review
finding. O-20 remains open and untouched. A reopen path is recorded for the trim: a GitHub-hosted
primary consumer would require amending D-02 first, and the key would be re-added in that same
change. N-19 is a reconciliation note rather than a decision; two of its clauses were corrected in
place under the owner's merge ruling, and its ID was neither renumbered nor retired.

## Notes not captured elsewhere

- **The fact-check round is what the arena bought.** Round 2 obliged each reviewer to verify the
  other's load-bearing claims rather than only argue them, and that duty reversed a position both
  sides had accepted: Fable's round-1 answer on R1.3 rested on Vitest targets being inferred by
  `@nx/vite`, and Sol refuted it with Nx 23's plugin split — inference moved to a separate
  `@nx/vitest` plugin. Fable's *mechanism* (a plugin-contributed target still counts as explicit)
  survived under the corrected plugin name, and both reviewers converged on one synthesis. A single
  reviewer would have shipped the wrong plugin name into R1.3.
- **A deadlock is a real outcome, not a failure to decide.** Q3's substance was unanimous in round
  1; only the wording of the roadmap's own Phase-0 healthchecks bullet stayed contested, 1-1,
  through rounds 2 and 3. Rather than break the tie by fiat, it was escalated, and the owner chose
  the achieved-state rewording — Sol's reading. That one bullet is the only roadmap change.
- **A parallel session collided with this one, and the owner ruled merge.** While the arena ran, a
  separate session applied a broader pre-implementation reconciliation to the same spec files, the
  ledger and the board, and its work was committed as `08ca51e` before these verdicts were applied.
  The collision was escalated rather than resolved locally. The owner ruled **merge**: that
  session's promotion of every §10 carryover is the base, and the arena's verdicts overlay it where
  the two disagree — which they did on exactly two points, Terminus's status and the size of the
  secret inventory. Both were corrected in N-19 and in review.md §12's affected rows; nothing else
  from the reconciliation was reverted.
- **Arena working papers are session-scratchpad only.** The round transcripts, per-reviewer
  position files, and cross-examination notes were never written into the repository; this record
  and review.md §13 are the durable artifacts.

## Rounds and verdicts

Round 1 asked both reviewers the same four questions independently. Round 2 gave each the other's
positions with a standing fact-check duty. Round 3 carried forward only the point still contested.

| id | Question | Verdict |
|---|---|---|
| Q1 | NF-01 / IC-01 — flat `{status, database}` contract or the Terminus envelope | **(a) flat contract**, unanimous in round 1; Terminus removed from the response path and the dependency set, the injected Drizzle probe seam kept |
| Q2 | R1.3 — what "explicit target" means and what T05 must create | **Synthesis**, converged in round 2 after the fact-check reversal: explicit = present in Nx's resolved configuration, `test` from a registered `@nx/vitest` plus per-project config, everything else manually declared |
| Q3 | L5 / L10 — identify versus create the two healthchecks.io checks | **Ensure-exists**, substance unanimous in round 1; the roadmap bullet's wording deadlocked 1-1 and the owner settled it as achieved-state rewording |
| Q4 | NEW-01 — keep or trim `HEALTHCHECKS_IO_PRIMARY_KEY` | **Trim** to a six-name inventory, unanimous in round 1; the name stays only where it is the thing being forbidden |

## Next step

Implementation starts at **T01** and proceeds in task order. No pre-implementation item remains
open. **O-20** remains a Phase-1 backup-lane entry condition owned by PT-005 and does not block
Phase 0.
