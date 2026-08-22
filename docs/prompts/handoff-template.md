# Template — handoff.md (implementer → next season)

> Lives at `.kiro/specs/PT-###-<slug>/handoff.md`. Updated by every implementer session.
> No-Fiction: real commands, real outputs, real gaps. An empty "deviations" section is a claim —
> make sure it is true.

```markdown
# Handoff — PT-### <title>

## Session <n> — <date>

### What I did
- <task ids ticked, one line each>

### Deviations from design.md (and WHY)
- <none | design said X, I did Y because Z — flagged for informed review>

### Decision gaps hit
- <none | question that needs the owner/planner, and what I did instead (stopped / minimal stub)>

### Known gaps / debt
- <not-covered edge case, skipped test, TODO left — with location>

### Commands run + real output (trimmed)
- `pnpm nx test <project> -- <filter>` → <pass/fail summary, pasted>

### Files touched
- <path list>

### Commit range
- `<baseSHA>..<headSHA>` — first to last commit of this task/story on `dev`; this is the exact
  diff input for the story's independent review (D-19; no branches exist to infer it from)

### Suggested next step
- <one line>
```
