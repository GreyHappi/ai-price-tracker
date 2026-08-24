# Season Record — 2026-08-22 #09 — Spike-review reconciliation

- **Type:** PT-001 correction and delegated-decision round
- **Operator:** GreyHappi · **Assistant:** Codex

## Goal

Apply the reconciled dual-review corrections to the PT-001 spike artifacts and propagate the six
accepted delegated decisions without changing owner-locked scope or ledger numbering.

## Inputs

- [PT-001 spike report](../spike-results.md) and its live re-check evidence
- Dual independent review positions supplied for Claude Fable xhigh and GPT 5.6 Sol Max — that was
  the **initial** dual review's lineup; from round 2 onward the same agreement loop ran with
  Claude Opus 5 xhigh and GPT 5.6 Sol xhigh (one continuous loop, changed model lineup)
- [Decision ledger](../../decision-ledger.md), [watchlist](../../product/watchlist.md), and
  [roadmap](../../roadmap.md)

## Outputs

- [Corrected spike report](../spike-results.md)
- [Ledger amendments and decisions](../../decision-ledger.md)
- [Updated watchlist](../../product/watchlist.md), [roadmap](../../roadmap.md),
  [CONTEXT invariants](../../CONTEXT.md), and [canonical plan](../../canonical-plan.md)
- [Board status](../../work/board.md): PT-001 was held **In Review** while the agreement loop ran;
  the board moves it to **Done** at loop close

## Decisions touched

O-14..O-19 · N-18 (item 4) · D-12 amendment · D-07 and O-08 annotations

## Notes not captured elsewhere

- **Position-reversal log:** Sol changed GATE fail → pass-with-corrections and watchlist-claim P1 →
  P2 after cross-examination. Fable changed pass → pass-with-corrections, softened unconditional
  pilot ENDORSE to preferred-candidate-with-five-conditions, and conceded the missed
  `/add`-representation P1. The agreement loop gates the commit; the final verdict is recorded at
  commit time.
- **Other notes not in N-18:** the round preserved the arabam category while blocking its adapter,
  added the Toyota canary, approved the exact BMW host, and retained no prior ledger ID by
  renumbering or deletion.

## Review rounds

Review rounds 1–4 ran with fix rounds between them. Round 2 (Opus xhigh empirical and Sol xhigh)
found 1 host-compat regression, 2 schema gaps (selector, `in_stock` tri-state), and consistency
fixes. Round 3 ended Opus pass / Sol fail on snapshot-storage coherence. The round-4 findings
(snapshot gitignore, the Phase-1 gate evidence bullet, snapshot retention, the source-entry
selector invariant, and wording nits) were fixed after round 4. Round 5 ended Opus pass / Sol
fail on one item (the age-only snapshot sweep could delete the newest snapshot); six surgical
edits followed, and round 6 (narrow closure check, sweep semantics fixture-tested) ended
**Opus pass / Sol pass — the agreement loop closed at round 6 with a unanimous GATE: pass.**

Round 7 ran **after** that close, on owner initiative: an independent triple review of the same
staged docs (GPT 5.6 Sol xhigh, Claude Fable xhigh, Claude Opus 5 xhigh). It returned two P1
classes — the dual-price guard missing for `list_price_minor` (both campaign presence and swing),
and snapshot evidence-chain retention together with how the Phase-2 gate accounts for missing
evidence — plus operational findings on the Windows runbook and cross-document consistency
findings. The orchestrator triaged them, and the accepted fixes were applied before commit — that
is this round.

Round 8 verified the applied round-7 fixes (Claude Opus 5, fresh-session independent-changes
prompt, no design/handoff/chat inputs): 15 of its 16 findings confirmed closed in the files, one
(`verify-full`) confirmed declined with a recorded rationale and revisit trigger, and the snapshot
sweep re-verified by an independent fixture run under both Windows PowerShell 5.1 and PowerShell 7
— including the all-files-older-than-35-days case that failed at round 5. It returned **GATE:
pass** with two residuals, both downstream of Phase 0: the phase gates never gained evidence items
for the invariants round 7 added, and the snapshot layout contract lived only in the runbook. Both
were fixed before commit — Phase-1 gate items for the suspicious/held snapshot, keep-two rotation
and url-coalescing, a Phase-2 gate item for the campaign-flip hold, and the layout contract
cross-referenced into CONTEXT and D-12.

Round 9 (2026-08-24, Claude Fable 5 session, operator-triaged): a GPT 5.6 Sol independent review
of the +849 staged snapshot returned three P1s and one P2, verdict pass-with-P1s. Disposition:
(1) backup evidence-chain P1 **accepted** — a backup-lane change whose predecessor change was also
backup-lane and >30 days old lost its before-evidence to artifact expiry; fixed by primary-side
ingestion of backup snapshot artifacts into the local `snapshots/` store (the reviewer's
bundle-the-predecessor proposal was infeasible: the ephemeral Actions runner never holds the
predecessor), with a Phase-1 gate item and a D-12 accepted-risk note for a >30-day primary outage.
(2) url-coalescing P1 **accepted** — the reviewed +849 snapshot's gate did already carry the
two-entry/two-lane criterion, added in round 8 (verified in round 11 against that snapshot's roadmap
blob `42d2ddda`, lines 81–82). What it still said was that concurrent calls claim "the entry" once,
leaving the atomic url-group claim unit, its `fetch_leases` fence, and the
member-becoming-due-mid-fetch case unpinned. (3)
verify-full P1 **half-confirmed**: the "libpq ≥ 16" revisit trigger was already met
(the client is postgres:17's libpq 17), yet the reviewer's `sslrootcert=system` fix fails in that
image — verified empirically on 2026-08-24: the postgres:17 image ships no `ca-certificates` — so
both runbook scripts now escalate to `verify-full` against a pinned ISRG Root X1 (Neon's
documented CA) and board item PT-004's ops-doc scope closes. (4) `scrape_runs` kind-column P2 **fixed** with a
pinned CHECK. The same session earlier aligned the plan/D-06 guard summaries with ADR-0006, made
D-12 state the campaign-flip hold its citation promised, and seeded board rows PT-003/PT-004.

Round 10 (2026-08-24, GPT 5.6 Sol, owner-requested reconciliation of the Opus 5 disposition)
agreed that `scrape_runs.kind` was closed and empirically reproduced psql 17.11 plus the missing CA
bundle in the local `postgres:17` image. It did not accept three closure claims as mechanically
complete: (1) backup ingestion lacked authenticated discovery, idempotency, archive/envelope
validation, failure visibility, and capture-time-safe retention; its newest-two policy also let
newer held snapshots evict accepted baseline evidence and never required an initial-baseline
snapshot, while one artifact per scrape run did not map to a multi-group worker execution; (2) the
alternative wording ("consistent entry lock ordering or a url lock") still left
the url fence undecided and contradicted D-02/D-26/ADR-0002's entry-only wording; (3) the TLS scripts
left `verify-ca` below hostname verification and trusted any file at the CA path. The follow-up pins
a (`adapter_key`, canonical `url`) `fetch_leases` fence, a digest-validated, workflow-batched
`snapshot-v1` ingest contract with the >30-day risk stated conditionally,
state-aware UTC capture-time retention, mandatory initial-baseline evidence, primary and backup
evidence barriers so alerts cannot outrun snapshots, `verify-ca` → `verify-full`, and an exact
CA-file SHA-256 check. Those changes were deliberately left unstaged for owner review — the owner
staged them before round 11 — and they do not claim a completed live ops install before Phase 1
creates the actual worker, secrets, scheduled tasks, and Neon smoke evidence.

Round 11 (2026-08-24, Claude Opus 5, owner-requested review of the staged set together with the
round-9/round-10 exchange). The owner had already staged the round-10 corrections, so the review
covered all 13 staged files rather than one side of the argument. Independent empirical checks: the
9 runbook PowerShell blocks parse under PowerShell 7.6.5 and Windows PowerShell 5.1; the snapshot
sweep, run against a fixture whose `LastWriteTime` order is deliberately inverted, keeps exactly the
two newest `accepted` files by capture time plus every `held`/`pending` file, deletes the aged
`discardable` and unprotected `accepted` ones, and warns on an out-of-contract name — identically on
both hosts; `postgres:17` reports psql 17.11 with `ca-certificates` uninstalled and no
`/etc/ssl/certs/ca-certificates.crt`; the pinned CA hash matches the current
`letsencrypt.org/certs/isrgrootx1.pem` byte for byte (DER fingerprint `96BCEC06…`, valid to 2035);
`actions/upload-artifact` v7.0.1 resolves to the pinned commit `043fb46d…`, whose `action.yml`
carries every input and output the backup barrier relies on (`if-no-files-found`, `overwrite`,
`retention-days`, `artifact-id`, `artifact-digest`); every `D-`/`N-`/`O-` reference in docs resolves
to a defined ledger row; and the staged diff carries no credential material. `C:\ops`, the CA file
and the three scheduled tasks are absent on this machine, which confirms PT-004's split scope rather
than a completed install. Three corrections were applied: the round-9 account of the +849
coalescing gate (round 10 over-corrected a claim the git object store settles in round 9's favour),
the Phase-1 gate item missing for PT-004's deferred live half, and the §5 dead-man table's silence
about ingest-suppressed primary pings. One residual is left to the owner rather than decided here:
D-12 gives an unvalidatable backup artifact no bounded escape, so a corrupt or never-parseable
artifact can hold the primary check red for the rest of that artifact's lifetime; an
operator-acknowledged quarantine would change D-12, so it is proposed, not applied.

Round 12 (2026-08-24, GPT 5.6 Sol reconciliation of round 11, applied by Claude Opus 5). It
confirmed the round-11 history correction against blob `42d2ddda`, the git counts, the absent live
ops install and both file caps, then returned five residuals, all accepted. The D-12 escape was
recorded only in this season record, which is neither a decision nor a backlog source (D-28), so it
is now open item **O-20** with board row PT-005 and a Phase-1 backup-lane entry condition. The
quarantine sketch gained the concrete shape the owner will rule on: a separate snapshot-ingest
healthcheck so liveness and evidence health stay distinct signals — healthchecks.io's free plan
monitors 20 jobs, verified 2026-08-24, so $0 holds — an explicit `/fail` from an invalid artifact,
quarantine only on human approval and recorded with artifact id/digest, workflow run, error class,
attempts, timestamps and reason, quarantine never counted as ingested, and no automatic skip after N
attempts. The §5 dead-man rows now read "primary unavailable **or** unhealthy" themselves instead of
being corrected by a note beneath them, and that note's wrong "§4 step 4" self-reference (step 4 is
the ping being diagnosed) became steps 1–3. The Phase-1 gate now demands the expected action,
trigger and principal plus a recorded smoke result rather than a task name that merely exists, and
the retention task gained its missing smoke step while the gate began requiring `/XML` verification
for all three tasks. The board's PT-001 label moved off round 10. Nothing here amends D-02 or D-12:
the escape stays proposed and owner-ratifiable.

Round 13 (2026-08-24, GPT 5.6 Sol final pre-commit check). The owner accepted the round-11/12
reconciliation with O-20 left open and owned by PT-005. The check found one proof/instruction gap:
the gate required all three scheduled tasks' XML definitions and successful smoke results, but only
the retention section showed an `/XML` query, and `/Run` alone proves launch rather than successful
completion. All three runbook sections now record `/XML` plus verbose task state after the smoke run
exits, with a zero last-run result required. PT-001 moves to **Done**; O-20 remains a Phase-1 entry
condition rather than an applied D-02/D-12 amendment.

## Next step

Give PT-002 Phase 0 its Kiro planner prompt. Settle **O-20** through PT-005 before the Phase-1
backup-evidence path goes live.
