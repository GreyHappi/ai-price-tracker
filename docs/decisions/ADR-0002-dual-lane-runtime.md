# ADR-0002 — Dual-lane runtime: home server primary + GitHub Actions backup

- **Status:** Accepted · **Date:** 2026-08-22 · **Ledger:** D-02, D-26, N-01, N-04

## Context
$0 infrastructure target. Scraping success depends heavily on IP reputation (residential IPs are
blocked far less than datacenter IPs). GitHub Actions scheduled workflows are best-effort:
queue jitter of minutes, occasional dropped runs, and public-repo scheduled workflows are
auto-disabled after 60 days of repo inactivity (warning e-mail + one-click re-enable).
Free PaaS options are effectively gone (Railway $1/mo credit, Render cold starts, Fly no free tier).

## Decision
1. **Primary:** home server (Docker) — in-process scheduler; every attempt, success or failure, is
   logged to `scrape_runs` with `lane='primary'`.
2. **Backup lane:** the same worker CLI on a 60-min Actions cron with `SCRAPE_LANE=backup`.
   It evaluates latest successful primary freshness **per adapter/source**, skips fresh sources
   (<3 h), and submits only stale/due work. A global "last primary run" is insufficient.
3. **Concurrency control:** both lanes atomically claim each due `source_entry` using an expiring
   PostgreSQL lease (`FOR UPDATE SKIP LOCKED` + `lease_owner`/`lease_until`). `lease_owner` is a
   fresh UUID per claim (a fencing token), never a stable process or lane id. The effect transaction
   must match that token and an unexpired lease; unique observation/event/delivery keys make crash
   retries replay-safe. DB timestamps alone are not concurrency control.
4. **Dead-man switches:** separate healthchecks.io checks. Primary pings after a successful
   cycle. Actions pings its own check after successful completion, including a correct no-op.
   Thus a disabled or failing Actions schedule is visible independently of primary health.
5. **Single scheduling surface:** one replay-safe `run-due-checks` command
   (claim → scrape → compare → persist → enqueue notification), called by the home scheduler,
   backup lane, and manual `/check`. No queue framework until a queue is real.

## Rejected alternatives
Actions as primary scheduler (lamport/grove/plan-2) · Railway/Render/Fly · a CF Workers
"standby bot" (impossible: Telegram polling XOR webhook, N-01) · BullMQ/Redis now.

## Consequences
(+) $0; residential IP for the hard sites; observable backup readiness; concurrent lane starts
cannot own the same source entry. (−) Bot commands are offline while primary is down (accepted
risk); the backup lane uses datacenter IPs and can cover only easy sources; expired leases and
at-least-once effects require explicit tests.

## Reopen signal
Primary uptime < ~95%/month, or external command access becomes a need → **migrate** (not
back up) the bot to a Cloudflare Workers webhook (ADR-0005's reopen path).
