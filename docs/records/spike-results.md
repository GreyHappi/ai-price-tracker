# Spike Results — PT-001 (Phase −1 risk spike)

- **Task:** PT-001 · **Date:** 2026-08-22 · **Operator:** GreyHappi
- **Method:** eight independent probes, one per endpoint, run by Opus 5 subagents against
  watchlist items where the source carries them (O-09), with source-level endpoint probes where
  necessary, synthesised here.
- **Conduct:** `robots.txt` fetched before every source; realistic desktop UA + `tr-TR`; spaced
  requests; **no CAPTCHA/login/anti-bot bypass attempted anywhere** (the no-bypass rule in
  [CONTEXT.md](../CONTEXT.md) under "Scraping").
- **Coverage:** probes targeted watchlist items where the source carries them. Vatan's
  `G614FR-S5061W` is an off-watchlist control proving extractability only, because Vatan carries
  none of the named watchlist SKUs. The watchlist named no concrete arabam listing, so an
  arbitrary live listing was probed. All 8 committed endpoints returned evidence. **No probe is
  NOT PROBED.**

## Evidence

| Source | URL | Format / rung | Sample or concrete blocker | Verdict |
|---|---|---|---|---|
| Vatan Bilgisayar — notebook | `https://www.vatanbilgisayar.com/asus-rog-strix-g16-9-nesil-ryzen-9-9955hx-rtx5070ti-12gb-32gb-1tb-16inc-w11.html` ¹ | server-rendered HTML; **rung 3** (JSON-LD has no `offers`, so rung 2 yields no stock) | G614FR-S5061W = **161.329 TL** (inline `dataLayer` → `items[0].price` = `"161329"`; DOM agrees); stock from the cart button | supported ¹ |
| ASUS E-Store TR — notebook | `https://odinapi.asus.com/recent-data/apiv2/ShopAPI/ShopFilterResult?CategoryName=&PageIndex=1&PageSize=100&ProductLevel1Code=laptops&SystemCode=asus&WebsiteCode=tr` ² | undocumented first-party JSON API; **rung 1** | `SalesModelName="G815LW-SA311-Gaming"` → `SortPrice="244999.000"`, `StockStatus="buy"`; 48 TR laptop SKUs in **one** GET | supported ³ |
| arabam.com — used-car listing | `https://www.arabam.com/ilan/sahibinden-satilik-toyota-rav4-2-0/39789362` | undetermined — no listing page was ever served | HTTP **403** `Cf-Mitigated: challenge`, `cType:'managed'` on every `/ilan/` + `/ikinci-el/` page; homepage and robots.txt: 200 on the *identical* client | **unsupported** |
| Kia TR — Sportage | `https://www.kia.com/tr/modeller/sportage/fiyat-listesi.html` | server-rendered table, prices in `data-*` attrs; **rung 3** | Live 1.6 DCT: DOM `data-turnkeyprice="2.249.000"` / `data-campaignprice="2.099.000"` (the DOM lowercases the source attributes); 6 trims; `If-Modified-Since` → **304, 0 bytes** | supported |
| Toyota TR — RAV4 | `https://turkiye.toyota.com.tr/middle/fiyat-listesi/fiyat_v3.xml` | static XML feed; **rung 1** | mechanism proven on siblings (Prado `ListeFiyati1` = `"19620000 TL"`); **RAV4 publishes no price** — its whole `<Model>` block is XML-commented, price elements empty | source parseable; target currently publishes no price ⁴ |
| VW TR — Tayron | `https://binekarac2.vw.com.tr/app/local/fiyatlardata/fiyatlar666.json` | first-party static JSON; **rung 1** | Tayron Life 1.5 eTSI DSG = **₺3.776.000,00** (`Aktif=="1"` group, `-ModelId "200"`); 6 trims + tax breakdown | supported |
| Škoda TR — Kodiaq | `https://www.skoda.com.tr/fiyat-listesi` | Next.js `__NEXT_DATA__` embedded state; **rung 2** | `vehicleKey=="kodiaq"` → Premium 1.5 TSI mHEV = **₺4.049.900**; Prestige campaign ₺3.998.100 vs list ₺4.749.900; 6 trims | supported |
| BMW TR — X3 | `https://borusanoto.bmw.com.tr/fiyat-listesi` | server-rendered div-table; **rung 3** | X3 20 X-Line = **6.853.400 TL**, X3 20 M Sport = **7.316.600 TL** ("Azami Anahtar Teslim Satış Fiyatı") | supported — Borusan Otomotiv BMW price list (source approved via delegated O-16 (owner-ratifiable)) ⁵ |

¹ Full URL: `https://www.vatanbilgisayar.com/asus-rog-strix-g16-9-nesil-ryzen-9-9955hx-rtx5070ti-12gb-32gb-1tb-16inc-w11.html`.
Vatan's search (`/arama/*`) and all `?page=`/`?srt=`/`?opf=` URLs are robots-disallowed, so
SKU→URL resolution must use `/sitemap.axd` or owner-pasted URLs, never site search.
² Full query: `https://odinapi.asus.com/recent-data/apiv2/ShopAPI/ShopFilterResult?CategoryName=&PageIndex=1&PageSize=100&ProductLevel1Code=laptops&SystemCode=asus&WebsiteCode=tr`;
currency/format from the companion `PriceConfig` call (`currencyCode:"TRY"`); pin/cache that
config so steady-state polling remains one request.
³ `shop.asus.com` is a separate, blocked host · ⁴ source supported, *target* has no published price
· ⁵ `www.bmw.com.tr` is not the working path — all three explained in Findings §2.

## Findings

### 1. Notebook pilot (O-08): **ASUS E-Store**, the **preferred candidate with five conditions**

The evidence makes ASUS the **preferred candidate**, not a proven winner. ASUS is rung 1 where
Vatan is rung 3: one unauthenticated GET returns the whole 48-SKU Turkish laptop catalogue with a
per-SKU price, so N watchlist laptops cost **one** request and `parse()` is a network-free
JSON→Observation map with a ready fixture. `StockStatus` is an observed two-value set (`buy` /
`notify_me`), not a documented enum; the adapter must fail loudly on an unseen value. Vatan needs a
~495 KB HTML fetch **per product per poll**, sends neither `ETag` nor `Last-Modified`, forces the
≥60-min Cheerio floor, and reads stock from a button class whose enum we observed only partially.
The decisive non-technical point is watchlist coverage: ASUS TR carries **one of the four named
SKUs** — `G815LW-SA311` at 244,999 TL, in stock, spec-matched, under the owner's ≤250,000 TL target
— while Vatan carries **none**, only spec-adjacent `-W-Gaming` Windows variants of the FreeDOS
SKUs the watchlist names. The confirmed pilot row is ASUS E-Store,
`SalesModelName="G815LW-SA311-Gaming"`. Phase 1 needs one *real* watchlist notebook target
(O-09). Of the five conditions, four are blocking before Phase 1 commits — (b) has a documented
provisional path in the roadmap:

(a) `ShopFilterResult` returns 200 from a real GitHub Actions runner; a challenge there means
`unsupported` on that lane, not a workaround.

(b) On a discounted SKU, the payable-price field is confirmed: `Price`/`RegularPrice` are empty,
only `SortPrice` is populated, and `SortPrice` is not a family "starting at" value (the roadmap's
documented provisional `SortPrice` path applies until the first discounted SKU is observed).

(c) Absence semantics follow O-19: an exhaustive unfiltered response is required;
inconclusive responses leave the baseline untouched; an exhaustive miss is a delisting candidate
only after a second complete run confirms it.

(d) Entry identity is catalogue URL + stable `SalesModelName` selector; the `-Gaming` suffix is
confirmed against the owner SKU, not silently treated as a different product.

(e) Exact TRY-to-kuruş bigint parsing is proven across the observed formats (`"161329"`,
`"2.249.000"`, `"244999.000"`, and `"3.776.000,00"` with a lira sign); no floats.

### 2. Unsupported and unclear findings (recorded honestly — nothing waved through)

- **arabam.com — direct listing pages unsupported; the source remains unsupported unless a
  permitted/authorized rung-0 route is found.** A Cloudflare *managed* challenge sits on exactly
  the data-bearing paths while the homepage and `robots.txt` serve 200 to the same client, so it is
  deliberate and path-scoped, not IP reputation alone — a datacenter runner would fare worse, not
  better. Playwright is not inherently circumvention: it remains a legal rung for ordinary JS
  rendering. Using it specifically to cross a managed challenge is circumvention. The paid arabam
  scraper providers examined (Carapis/Apify/ScrapingBee) would outsource that challenge-crossing
  circumvention and would fail the $0 target, so they are not adopted.
  **Impact on D-07:** the used-car category remains in MVP, but Phase 2 item 1 is blocked and no
  adapter is built while arabam is unsupported; the MVP exit gate records it as
  unsupported-blocked rather than silently dropping it. sahibinden.com is *not* an equivalent
  stand-in (already Phase-4-deferred behind three conditions, known for anti-bot + a login wall).
  A minimal reopen probe is one GET checking for `Cf-Mitigated`.
- **`shop.asus.com` — `unsupported` host, not an unsupported source.** Its `robots.txt` itself
  returns 403 behind DataDome, so the whole host is off-limits and was not probed further — which
  does **not** contaminate the ASUS verdict, since the working path is `odinapi.asus.com` (robots
  response: 302 to `/Error/404`, an allow-all effect; no anti-bot). The adapter must allowlist
  `odinapi.asus.com` and deny both
  `shop.asus.com` and `www.asus.com` faceted-filter URLs (robots-disallowed).
- **BMW — supported, but not at the URL the watchlist implies.** `www.bmw.com.tr`'s price page
  carries zero prices; its data sits in an iframe to `www.borusanotomotiv.com/bmw/…`, which is
  **robots-disallowed** and was not fetched. Prices came from the importer's BMW-branded dealer
  subdomain `borusanoto.bmw.com.tr` (robots `Allow: /`) with the official "Azami Anahtar Teslim"
  figures — a scope question for owner sign-off, not a technical gap.
- **Toyota RAV4 — source supported, target has no data.** `fiyat_v3.xml` is one of the cleanest
  paths found (rung 1, verified 304). But RAV4 is mid-generation-changeover: its model block is
  XML-commented with empty price elements and the model page redirects to "yeni-rav4-yakinda", so
  the watchlist row produces **no observation** until Toyota republishes. O-15 keeps RAV4 as
  the live test of the "target exists, no price published" path and adds a temporary priced
  Corolla Cross canary from the same feed to prove ordinary extraction; archive, never delete, the
  canary when RAV4 pricing returns. O-06 is not reopened.
- **Watchlist SKU gap (D-29 / Phase 4).** Three of the four named ASUS SKUs (`G614FR-S5112`,
  `G614FR-S5132`, `G815LR-TT362`) exist at *neither* notebook candidate — product scope, not
  extractability, but direct evidence for the Phase-4 SKU-discovery epic. They are discovery
  candidates, not currently purchasable at a committed source. The open owner item is to propose
  exact replacements from a fresh ASUS catalogue and obtain human approval before tracking;
  spec-equivalent silent binding is rejected because identity corruption can evade the swing guard.

### 3. Risks and observations for the Phase-1 walking skeleton

- **Several assumptions remain unproven:** (1) a real GitHub Actions runner can fetch the ASUS
  endpoint; (2) `SortPrice` is the payable price on a discounted SKU, not a family "starting at"
  value; (3) exhaustive-response/missing-SKU semantics can distinguish inconclusive transport from
  a true miss; (4) `SalesModelName` and the `-Gaming` suffix preserve the owner SKU's identity; (5)
  every observed TRY number format parses exactly to kuruş; and (6) OEM campaign/list precedence
  and null availability remain stable. Every probe ran from a Turkish residential IP; Vatan and
  Kia both sit behind Cloudflare with no challenge issued, and arabam shows what one looks like
  when it fires. Phase 1's backup lane is GitHub Actions — verify the pilot source from a real
  runner early, and treat a challenge there as `unsupported` on that lane.
- **Transport validators, body hashes, and semantic hashes are separate.** `ETag`/
  `If-Modified-Since` is not universally available: 304 was verified at Kia and Toyota,
  headers were present but unverified at VW, and they were **absent entirely** at Škoda, Vatan and
  BMW dealer. When validators are absent, a transport/body hash is only a fetch-efficiency
  fallback for skipping an unchanged response; it is not semantic change detection. Semantic
  comparison uses `snapshot_hash` over (`price_minor`, `list_price_minor`, `currency`,
  `in_stock`). Nor does any **10–15 min fast lane** qualify: the cheap rung 0–2 paths (ASUS, VW,
  Toyota, Škoda) are all *undocumented* internal endpoints with no published terms, so O-13's
  precondition is unmet and 60-min + jitter stands everywhere.
- **OEM sources publish two prices.** Kia, Toyota, VW and Škoda each expose a list price and a
  campaign price. D-12 now makes `price_minor` the effective advertised price (campaign when
  published, otherwise list), stores nullable `list_price_minor`, and hashes both fields, so a
  campaign start/end or list-price change is visible while target rules stay on the payable price.
- **TRY formats and the first baseline are hazardous.** Vatan, Kia, ASUS and VW expose different
  separators/decimals (`"161329"`, `"2.249.000"`, `"244999.000"`, `"3.776.000,00"` with a lira
  sign). A missing baseline on the first observation means a wrong first parse becomes the
  accepted baseline; the swing guard cannot catch that error. Parsing must be exact bigint logic,
  with no floats, and every source format needs a fixture.
- **A wrong-product bind is invisible to the sanity guard.** BMW's `line_*` row classes are stale
  copy-pasted slugs: rows classed `line_yeni-bmw-x3-…` carry **X1** data, and X1/X3 prices sit
  within ~30%, so the >70% swing guard never catches it. `parse()` must assert product identity
  (title text, trim, engine), not just price plausibility.
- **Never infer availability from price presence.** On a Vatan sold-out page the main price block
  disappears but `#mobilePrice` still renders a value — an adapter keyed on that alone reports a
  live offer for a sold-out item. Read stock from the stock control and fail loudly on an unseen
  state. OEM lists carry **no** availability at all: pin `in_stock` (`null`, not `true`) before the
  first OEM write; `null` participates in `snapshot_hash` as its own state (D-12).
- **Transient no-data must not read as a change.** Toyota's feed root carries `isUpdating`; in a
  publish window it flips to `"1"` and every model disappears. That must log to `scrape_runs` and
  leave the baseline alone, or each monthly update emits bogus `change_events`. Same class of bug
  as arabam's: a 403 must never be interpreted as "sold/delisted".
- **Windows transport hazard:** `www.bmw.com.tr` (Akamai) produced a TLS handshake/stack hang with
  schannel-curl on Windows (TLS 1.3 has no renegotiation); only `--tlsv1.2 --http1.1` completed.
  Node's fetch stack may behave differently. It is off the working path, but the primary lane *is*
  a Windows machine (`runbooks/windows-server.md`), so the shared fetch wrapper should surface
  handshake hangs as a distinct error class with a hard timeout.
- **Endpoint discovery is fragile in a different way at every source:** Kia's sitemap emits a
  generation-coded URL that 301s; VW's `fiyatlar666.json` is an opaque magic filename; Toyota's
  `fiyat_v3.xml` is version-numbered; Škoda's `_next/data/{buildId}` rotates every deploy — all
  nightly-live-smoke concerns, since fixtures catch code regressions only.
