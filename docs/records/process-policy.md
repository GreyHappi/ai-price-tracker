# Process Policy — intentionally omitted ceremonies

> The No-Fiction rule (D-18) says every artifact is generated from real data and forbids a document
> per non-event. This single file is the exception log: each ceremony deliberately skipped, why, and
> what covers the gap instead. **Append only** — never delete or rewrite a row; supersede it with a
> newer one. A ceremony that was simply forgotten does not belong here; it belongs in the season
> record as a miss.

| Date | Phase / task | Ceremony omitted | Rationale, and what covers the gap |
|---|---|---|---|
| 2026-08-24 | Phase −1 gate (PT-001) | Fresh-session **informed review** ([prompts/informed-review.md](../prompts/informed-review.md)) | Its inputs do not exist. PT-001 was a timeboxed probe season below the spec threshold (D-21): there is no spec triplet (`.kiro/` is empty), no `handoff.md`, no committed code (the repo tracks Markdown only) and no CI results, so all four of the prompt's questions — spec conformance, recorded deviations, missing error paths, invariants in touched code — are unanswerable, and its `review.md` destination has no story to attach to. Running it would manufacture exactly the fictional artifact D-18 forbids. The gate is carried instead by the Phase −1 criterion itself (the 8-endpoint evidence table in [spike-results.md](spike-results.md)) and by the independent half of D-19: an independent triple review (round 7), a fresh-session independent-changes review (round 8), and reconciliation rounds 9–12 — all logged in [season #09](seasons/2026-08-22-09-spike-review-reconciliation.md). Phase 0 produces real specs and code, so its gate runs **both** prompts as written. |
