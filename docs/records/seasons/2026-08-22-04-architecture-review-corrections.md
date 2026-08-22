# Season Record — 2026-08-22 #04 — Architecture review corrections

- **Type:** independent review of Fable merge + external architecture document
- **Operator:** GreyHappi
- **Assistant:** Codex/GPT
- **Next reviewer:** Claude (Fable 5)

## Goal

Accept compatible external-architecture ideas while correcting unproven assumptions and unsafe
runbook commands. Preserve owner-locked five-OEM and thin-WebSocket scope.

## Inputs

- Fable season #03 report and Opus runbook/audit transcript
- `stock-price-tracker-final-architecture.md` (2026-08-20)
- Current staged documentation set
- Current official Tailscale, Neon, and PostgreSQL documentation

## Outputs

- Updated [decision ledger](../../decision-ledger.md), [canonical plan](../../canonical-plan.md),
  [CONTEXT](../../CONTEXT.md), [roadmap](../../roadmap.md), [watchlist](../../product/watchlist.md),
  [board](../../work/board.md), and [Windows runbook](../../runbooks/windows-server.md)

## Decisions touched

D-06, D-07, D-29, D-30; N-14, N-15; O-08, O-09, O-12, O-13.

## Notes not captured elsewhere

- Accepted: deferred board automation, real watchlist, Phase-4 discovery/dealer depth, extensible
  events, database constraints, desktop niceties, analytics catalog, and per-source intervals.
- Corrected: all five MVP OEM endpoints must be probed; MVP exit covers all seven selected
  sources; seller text is a snapshot rather than identity; WS needs an actual-host resilience
  gate; current Tailscale CLI and failure-safe Neon backup flow replace the original runbook text.
- Existing records remain unchanged because season history is append-only.

## Next step

Fable reviews the unstaged correction diff. After reconciliation, begin PT-001 Phase −1 spike.
