# Board

> **Single manual status source** (O-07): edit this file by hand; do NOT duplicate status in
> story front-matter. Trigger for automation: ≥5 active stories OR the first status-drift
> incident — then status moves into front-matter and `tools/board.ts` + lefthook generate this
> file (with `pm-check` in CI). WIP limit: **1** (In Progress).

| Backlog | Ready | In Progress (≤1) | In Review | Done |
|---|---|---|---|---|
| | PT-002 Phase 0 foundation — spec triplet accepted 2026-08-24 after review + regeneration (`review.md` §9), second opinion applied 2026-08-25 (§11: IC-02 and the D10 parser resolved, NEW-01 refuted); spec season closed, **cleared to start at T01**. Decide NF-01 before T17/T18 and R1.3 at T05; eleven §10 items still open | | | PT-001 Phase −1 risk spike (review reconciliation accepted) |
| PT-003 ops: extract shared parse-neon-url.ps1; backup script + §5 audit snippet dot-source it — **folded into the PT-002 spec as T29** (2026-08-24, board instruction discharged); the deliberate runbook §3/§5 duplication stays until T29 runs | | | | |
| | | | | PT-004 ops-doc: Neon backup TLS design → verify-full (runbook closed and empirically image-tested; live install/smoke belongs to Phase 1) |
| PT-005 ops: settle **O-20** — bounded escape for an unvalidatable backup-snapshot artifact (separate ingest healthcheck + human-approved quarantine); owner decision first, then the D-02/D-12 amendment. Phase-1 backup-lane entry condition | | | | |

_Seeded 2026-08-22. First real population continues with Phase 0._
