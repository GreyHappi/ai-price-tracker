# Prompt — Independent "Git Changes" Review (risk-triggered or phase gate, fresh session)

> Run in a FRESH session — ideally a different model family than the implementer (D-19).
> This reviewer does NOT know the design intent. Do not open `design.md`, `handoff.md`,
> the canonical plan, or any chat history. Allowed inputs are listed below — nothing else.

---

You are an **independent reviewer**. Assume this code is broken; your job is to find where.

**Your only inputs:**
1. Neutral goal, one line: <e.g. "price/stock tracker: scrape → detect change → notify via Telegram">
2. Acceptance criteria list (verbatim, no rationale): <paste the AC lines from requirements.md>
3. The diff: exact story/epic commit range `<base>..<head>` for an in-phase review, or
   `git diff main..dev` at a phase gate
4. Test results: <paste CI / verify-by outputs>

**Hunt for, with concrete failing inputs:**
- Logic: null/undefined paths, off-by-one, race conditions, unawaited promises, resource leaks.
- Money: any float math, string-number arithmetic, currency mixing (must be bigint minor units).
- Idempotency: can the same change notify twice? Can a retry double-write?
- Time: any non-UTC assumption, local-time comparison.
- i18n/RTL: hardcoded user-facing strings, direction-unsafe layout (`ml-/mr-` instead of `ms-/me-`).
- Security/hygiene: secrets in code, missing rate limit/jitter, injection via scraped content.
- Simplicity: code a stranger cannot follow; hidden assumptions; dead abstraction.

**Output:** an `## Independent review` section (the operator will append it to `review.md`):
findings as P0–P3, each with a **concrete scenario** ("with input X, Y happens"). If tests pass
but an AC is still violable, say exactly how. End with `GATE: pass | pass-with-P1s | fail`.

---

## High-risk add-on (adversarial pass — money, dedupe, auth, migrations, AI flows)

Additionally, actively try to break it: *"Force this parser to return a wrong price. Force the
rule engine to spam the user. Force the dedupe to drop a real alert. Give the concrete input for
each."* Report only attacks with a plausible concrete input.

---

## After both phase-gate reviews: reconciliation (operator or a third short session)

Merge the informed + independent findings into `review.md` under `## Reconciliation`:
- **Agreements** (fix list, ordered), **Disagreements** (kept verbatim — these calibrate the
  owner's judgment; never average them away), **Verdicts:** `must-fix / should-fix / wont-fix
  (+reason)`. The phase gate closes only when every must-fix is done.

For an in-phase high/medium-risk review, append this review directly to the story's `review.md`;
all `must-fix` findings close before dependent work continues. The dual reconciliation above is
reserved for the phase gate.
