# ADR-0005 — Telegram bot: grammY, long-polling

- **Status:** Accepted · **Date:** 2026-08-22 · **Ledger:** D-03, N-01

## Context
The primary runtime is a home server behind NAT with no public URL (ADR-0002). Telegram is the
primary notification and command channel.

## Decision
grammY in long-polling mode, running as a module inside `apps/api`.

## Rationale
Long-polling needs no port forwarding, DDNS, reverse proxy, or TLS — ideal for a NAT'ed home
server. grammY is TS-first with the active plugin ecosystem (Telegraf's has migrated to it).

**Key technical fact (N-01):** a Telegram bot is either polling or webhook — `setWebhook`
disables `getUpdates`. Therefore a serverless webhook cannot exist as a *standby* command
receiver next to the polling primary. The backup lane (ADR-0002) covers scrape+notify only,
via direct `sendMessage` — no webhook required. Commands being unavailable while the primary is
down is an accepted risk.

## Rejected alternatives
- Webhook on Cloudflare Workers now (recursive-lamport) / on Railway (plan-2): requires public
  hosting and forfeits the polling simplicity; kept as the future migration target.
- Telegraf: less active than grammY.

## Consequences
(+) Zero network setup. (−) No commands during primary downtime.

## Reopen signal
Primary uptime < ~95%/month or external command access needed → migrate the bot wholly to a
Cloudflare Workers webhook (federated-grove's Phase-2 design).
