# Watchlist (seed)

> Owner-approved initial tracking targets (O-09; source: external architecture doc, 2026-08-20);
> O-17 authorises the active-list restructure. Living seed data — edit freely. The Phase −1 spike
> samples every committed source using these targets where that source carries them; the walking
> skeleton uses one real notebook target from the spike-selected preferred source.
> Prices are owner targets, not predictions. Money rule D-11 applies in the system (bigint minor
> units); TL figures here are for readability.

## Laptops (ASUS ROG)

| SKU / Model | Key specs | Target price | Committed source / status |
|---|---|---|---|
| ROG Strix G18 G815LW-SA311 | Intel Core Ultra 9 290HX Plus · RTX 5080 16 GB · 32 GB · 18" · Mini-LED / Nebula HDR | ≤ 250,000 TL | **ASUS E-Store TR — confirmed in stock, under target; `SalesModelName="G815LW-SA311-Gaming"`** |

Candidate notebook sources: **ASUS E-Store** and **Vatan** — ASUS is the Phase −1 preferred pilot
candidate, subject to the five Phase-1 entry conditions recorded in
[docs/roadmap.md](../roadmap.md) Phase 1 (O-08). Marketplaces (Trendyol / Hepsiburada) stay
deferred (D-06/D-07).

## Discovery candidates (exact SKU unknown, or not currently purchasable at a committed source)

These rows are retained as Phase-4 SKU-discovery evidence, not active tracking targets:

| SKU / Model | Evidence / status |
|---|---|
| ROG Strix G18 G815LR-TT362 | Absent from both committed notebook candidates; discovery candidate only. |
| ROG Strix G16 G614FR-S5132 | Absent from both committed notebook candidates; discovery candidate only. |
| ROG Strix G16 G614FR-S5112 | Absent from both committed notebook candidates; discovery candidate only. |
| ROG Strix G16 — Ryzen 9 8940HX + RTX 5070 Ti + 32 GB | Exact Turkish SKUs unknown; discovery-dependent. |

Exact replacement products will be proposed from a fresh ASUS catalogue and require human
approval before tracking (open owner item). Spec-equivalent silent binding is rejected: identity
corruption can evade the swing guard.

## Cars — new-car OEM price lists (five brands in MVP, O-10)

| Model | Source | MVP tracking |
|---|---|---|
| Kia Sportage | Kia TR price list | list/campaign price — 1st OEM adapter |
| Toyota RAV4 | Toyota TR price list (`fiyat_v3.xml`) | successful run, no observation, and no false delisted/out-of-stock event; target-exists/no-published-price live test |
| Toyota Corolla Cross | Toyota TR price list (`fiyat_v3.xml`) | **temporary priced canary** for ordinary extraction; archive, never delete, when RAV4 pricing returns |
| VW Tayron | Volkswagen TR price list | list/campaign price |
| Škoda Kodiaq | Škoda TR price list | list/campaign price |
| BMW X3 | **Borusan Otomotiv BMW price list** | list/campaign price; approved via delegated decision O-16 (owner-ratifiable), host `borusanoto.bmw.com.tr` |

Trim/dealer/city/color/stock-count depth and new-trim discovery: **Phase 4 epic** (O-11, D-29).

## Used cars

arabam.com — single-listing URLs (separate category retained in MVP, D-07); source currently
**unsupported-blocked** pending a permitted/authorized first-party route, so no adapter is built.
A minimal periodic reopen probe may issue one GET checking `Cf-Mitigated`; category/saved-search
tracking remains part of the Phase 4 epic.
