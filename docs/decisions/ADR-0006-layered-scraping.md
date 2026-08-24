# ADR-0006 — Scraping: layered ladder, Playwright last

- **Status:** Accepted · **Date:** 2026-08-22 · **Ledger:** D-06, D-07, N-03, N-09, O-08, O-14

## Context
Scraping is the project's riskiest subsystem: sites change HTML silently, anti-bot systems block
datacenter IPs aggressively, and $0 infrastructure rules out proxy stacks. Pilots: Vatan
(notebook), arabam.com (used car), Kia TR + Toyota TR price lists (PDF-capable).

## Amendment — 2026-08-22 spike outcome

The preferred notebook pilot candidate per the spike is **ASUS E-Store**; Vatan remains the
alternate. ASUS is subject to O-08's five Phase-1 entry conditions. `arabam.com` is
**unsupported-blocked** behind a Cloudflare managed challenge (O-14), joining `sahibinden.com` as
an anti-bot-blocked source under the no-bypass rule.

## Decision
Extraction ladder, cheapest first — **0)** official API/feed → **1)** native fetch →
**2)** JSON-LD / meta / embedded state → **3)** site adapter + Cheerio → **4)** PDF (`unpdf`) →
**5)** Playwright only when forced.

- `SourceAdapter` port with a **pure** `parse()` (recorded input → Observation; fixture-tested).
- Hygiene: robots.txt, per-domain rate limit + jitter, realistic UA, ETag/If-Modified-Since,
  consecutive-failure counter → auto-pause + operator alert, adapter `health_score`.
- Sanity guards: price ≤ 0 or > 70% swing on either price field, or a campaign-presence flip
  (D-12 amendment) ⇒ suspicious — hold the alert, ask the human.
- **AI extraction quarantined:** `extraction_method='ai'` + confidence; never alerts
  unconfirmed; primarily drafts parser repairs (human-approved).
- **Test pairing (N-03):** fixture tests catch code regressions only — they cannot see live-site
  changes; the nightly live-smoke job (1 real URL/adapter, non-CI-blocking, failure → Telegram)
  catches drift.
- **Never bypass** CAPTCHA/login/anti-bot; such sources are marked `unsupported`.
  sahibinden.com remains deferred until all D-07 conditions pass: accepted proxy budget,
  1 month stable MVP, and documented terms/access review confirming the intended method.
- Crawlee deferred until category/list crawling is real.

## Rejected alternatives
Playwright-first (linear-lark) — 10–50× more expensive per check and doesn't beat anti-bot
anyway · Crawlee day 1 (tender-sketch, replicated-crayon) · stealth/proxy stack now
(replicated-crayon).

## Consequences
(+) Most checks are cheap and fast; honest "unsupported" list; home residential IP does the
heavy lifting. (−) Some sources stay out of reach until Phase 4 budget decisions.

## Reopen signal
List crawling ships → adopt Crawlee. sahibinden: all three D-07 budget, stability, and
terms/authorized-access conditions pass; otherwise it remains unsupported.
