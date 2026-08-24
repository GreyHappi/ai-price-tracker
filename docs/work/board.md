# Board

> **Single manual status source** (O-07): edit this file by hand; do NOT duplicate status in
> story front-matter. Trigger for automation: ≥5 active stories OR the first status-drift
> incident — then status moves into front-matter and `tools/board.ts` + lefthook generate this
> file (with `pm-check` in CI). WIP limit: **1** (In Progress).

| Backlog | Ready | In Progress (≤1) | In Review | Done |
|---|---|---|---|---|
| | PT-002 Phase 0 foundation | | | PT-001 Phase −1 risk spike (review reconciliation accepted) |
| PT-003 ops: extract shared parse-neon-url.ps1; backup script + §5 audit snippet dot-source it (fold into the PT-002 spec; duplication is deliberate until then — runbook §3) | | | | |
| | | | | PT-004 ops-doc: Neon backup TLS design → verify-full (runbook closed and empirically image-tested; live install/smoke belongs to Phase 1) |
| PT-005 ops: settle **O-20** — bounded escape for an unvalidatable backup-snapshot artifact (separate ingest healthcheck + human-approved quarantine); owner decision first, then the D-02/D-12 amendment. Phase-1 backup-lane entry condition | | | | |

_Seeded 2026-08-22. First real population continues with Phase 0._
