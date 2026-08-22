> **Historical raw session export — season #01 appendix (D-28).** This is an unedited transcript
> kept for process history, not a source of truth. Several positions quoted inside were later
> corrected or superseded in the ledger: early timestamp-based lane coordination → per-entry leases
> (D-02); UI-only i18n → EN/TR/AR across UI, bot, and notifications (D-23); a browser-bundled
> static token → Tailscale Serve identity with no browser secret (D-09/D-10); a fixed monthly AI
> budget → billing verification first (O-03); an immediately generated board → deferred automation
> (O-07). **`decision-ledger.md` and `CONTEXT.md` are authoritative.** Local absolute paths have
> been scrubbed to `[repo]`; nothing else in the transcript is rewritten (append-only, D-28).

A. Tam (veya tama yakın) konsensüs — detaylandırmıyorum
Şu konularda 8 plan da (konuya değinenlerin tamamı) aynı görüşte; tartışmaya gerek yok, doğrudan kabul edilebilir:

ORM = Drizzle (Prisma/TypeORM/MikroORM her planda reddedilmiş, gerekçeler de aynı).
NestJS modüler monolit, mikroservis yok.
Zod 4 + nestjs-zod, tek şema kaynağı (drizzle-zod → Zod → OpenAPI → FE tipleri zinciri).
Web/mobil için tek API, BFF ayrımı yok (YAGNI).
URI versiyonlama /api/v1 1. günden ("3 satır maliyet, retrofit pahalı").
MVP'de Redis/BullMQ yok; çoğu plan JobQueue port arayüzünü şimdi tanımlayıp implementasyonu erteliyor.
React 19 + Vite + TanStack Router/Query + shadcn/ui + Tailwind; resmi dashboard-01 bloğu başlangıç (birden çok plan bağımsızca doğrulamış).
Zustand (client UI) + TanStack Query (server state); server verisi Zustand'a kopyalanmaz.
Native fetch + ince tipli wrapper; axios yok.
grammY (Telegraf değil — adı geçen tüm planlarda).
Script dili TypeScript, Python sadece ileride ML/analiz gerekirse ayrı sınırdan.
Vitest her yerde (FE+BE tek runner) + Playwright E2E.
Tauri 2; Flutter reddedildi (ikinci codebase/dil maliyeti).
Landing page = ayrı Astro SSG, sonraya; Tauri CSR'ın SEO'ya uygun olmadığı teyidi her planda.
Hermes/OpenClaw ürün çekirdeğine girmez (jellyfish/lark: ileride "kaynak keşfi spike'ı" veya referans pattern olarak olabilir).
React'te kal, Angular/Svelte'e geçme (jellyfish'in güzel notu: NestJS zaten Angular'ın zihinsel modelinin ~%70'ini öğretiyor).
AI ürün ekleme: iki yol (direkt URL + serbest metin→AI çözümleme), insan onayı zorunlu, model sırası DB'de ve Telegram/dashboard'dan değiştirilebilir, fallback zinciri, çıktı Zod ile doğrulanır.
Model isim düzeltmesi: "Gemini 3.7 Flash-Lite" yok — Lite ayrı bir aile (planların çoğu 3.5 Flash-Lite diyor, sketch 3.1 diyor; plan-1'in tavsiyesi geçerli: isimler entegrasyon anında yeniden doğrulanır, zaten DB'de tutulacak).
Kiro-tarzı spec (requirements/design/tasks) + ADR (reddedilen alternatifli) + docs/ ailesi (runbooks, guides, gates, records).
Tek dev doküman anti-pattern — bu planlar repo'da bölünecek.
Sezonlar arası bilgi akışı dosyalardan, sohbet hafızasından asla.
Scaffolding kuralı: "yapıyı/portları iskele et, davranışı/özelliği etme" + aynı YAGNI listesi (auth mantığı, billing, RBAC, i18n, k8s, event bus şimdi yok).
user_id/owner_id kolonu 1. günden (derinliği tartışmalı — bkz. madde 13).
Adapter pattern (SiteAdapter/SourceAdapter) — yeni site = yeni adapter, çekirdek değişmez.
Fixture/golden-file adapter testleri (7/8 planda; sketch bunu "en değerli test türü" ilan ediyor, haklı).
Scraping görgü kuralları: robots.txt, rate limit, CAPTCHA/login aşılmaz, kişisel kullanım.
B. Ayrışan konular — kararını bekleyenler
1. Veritabanı: SQLite mi, Postgres mi, ne zaman?
Postgres 1. günden (5 plan): sketch (Docker, en sert gerekçe: Drizzle şemaları dialect'e özel — sqliteTable→pgTable geçişi gerçek yeniden yazım; SQLite tek-yazar kilidi SQLITE_BUSY — scraper+API+bot aynı anda yazacak; timestamptz/jsonb/partial index tip kaybı), lark, grove (Neon), jellyfish (Neon prod + Docker local dev — "enterprise'ın standart deseni, doğrusunu öğreniyorsun"), plan-2 (Neon).
SQLite ile başla (2 plan): plan-1 (WAL modu, geçişe "migration rehearsal" planıyla dürüst yaklaşım), lamport (Turso/libSQL — hosted SQLite, çünkü mimarisi GH Actions+Workers'tan erişilebilir DB istiyor).
İkisi birden (1 plan): crayon (dev=SQLite, prod=Neon; Drizzle multi-DB config baştan).
Önerim: Postgres 1. günden (Neon free tier + lokalde Docker). Sketch'in üç somut maliyeti çürütülemiyor; ayrıca "SQLite→Postgres sonra geçeriz" tam da planların uyardığı ertelenen-pahalı iş. Crayon'un dual-DB'si bile iki dialekt'i sürekli senkron tutma vergisi ödetir. SQLite'ın tek avantajı (sıfır servis) Neon free tier varken anlamsız.

2. Hosting + scraper zamanlayıcısı — en büyük mimari çatışma
GitHub Actions cron ana scheduler (3 plan): lamport (Actions scrape→Turso→Telegram), grove (jitter kabul edilebilir dedi), plan-2 (saatlik Actions worker, "best-effort, SLA değil" notuyla).
Kendi makine/ev sunucusu, in-process scheduler (4-5 plan): sketch ("GH Actions scheduler olarak kullanılamaz" — 10-30 dk gecikme, 60 gün commit yoksa sessizce kapanır, long-polling barındıramaz), jellyfish (aynı gerekçeler + asıl koz: ev IP'si = residential IP, datacenter IP'lerinin Cloudflare'de agresif bloklandığını iki plan bağımsızca söylüyor; Actions'ı Faz 2'de yedek hat olarak tutar), lark, plan-1, (crayon belirsiz-lokal).
Bulut PaaS (1 plan): plan-2 (Railway API + webhook; Railway free'nin $1/ay krediye düştüğünü jellyfish ve grove da teyit ediyor — pratikte ücretli demek).
Önerim: kendi makine/ev sunucusu (Docker) + Neon. Gerekçe zinciri: (a) scraping başarısı için residential IP gerçek avantaj — bu projenin en riskli parçası scraping, hosting kararı ona hizmet etmeli; (b) Actions'ın gecikme/60-gün/uyku sorunları bir "takip" ürünü için güven zedeler; (c) $0 hedefi korunur. Actions sadece CI. PC'nin kapalı olduğu saatler kabul edilen risk (jellyfish/plan-1 bunu açıkça kabul ediyor). Eğer "makinem hep açık olmasın" dersen ikinci en iyi: plan-2'nin Railway kurgusu (ama Hobby $5/ay'a hazır ol).

3. Telegram bot modu
Long-polling MVP (4 plan): sketch, jellyfish (port açma/DDNS/reverse-proxy yok — ev sunucusuyla mükemmel uyum), lark, plan-1.
Webhook baştan (2 plan): lamport (Cloudflare Workers), plan-2 (Railway).
Hibrit (1 plan): grove (dev polling → Faz 2'de CF Worker webhook).
Önerim: long-polling (madde 2'deki hosting kararının doğal sonucu). Webhook'a geçiş Faz 2+ konusu; grove'un CF Worker fikri o gün için en iyi aday.

4. Monorepo aracı
Nx (7 plan): lamport, sketch, lark, crayon, jellyfish, plan-1, plan-2. Sketch ve jellyfish enforce-module-boundaries lint kuralını "mimariyi çürümekten koruyan tek otomatik mekanizma" diye ayrıca vurguluyor.
pnpm + Turborepo (1 plan): grove ("Nx ağır config; enterprise CV değeri var ama geçiş mümkün").
Önerim: Nx. 7-1 çoğunluk + senin enterprise-öğrenme hedefin + module-boundary kuralları. Grove'un "geçiş her zaman mümkün" tesellisi doğru ama ters yönde de geçerli.

5. Scraping stratejisi ve motoru
Katmanlı merdiven, ucuzdan pahalıya (7 plan): hepsi benzer sıra — JSON-LD/schema.org → gömülü state (__NEXT_DATA__) → gizli JSON API → cheerio selector → Playwright son çare. Jellyfish en detaylısı (+PDF katmanı unpdf ile — OEM araba fiyat listeleri için) ve "fetch+cheerio, Playwright'tan 10-50× ucuz" diyor. Sketch resmi/affiliate API'yi katman 0 olarak ekliyor.
Playwright-first (1 plan): lark — gerekçesi hedef site seçimi: Trendyol/HB/Amazon zaten JS-ağır.
Crawlee framework'ü (2 plan): sketch (queue/autoscaling/session-pool bedavaya gelir, "Redis gerekmiyor" argümanının dayanağı), crayon (AdaptivePlaywrightCrawler — önce HTTP dener). Diğer 6 plan sade fetch + kendi adapter'ları (+p-limit).
Anti-bot derinliği: jellyfish en gerçekçisi — sahibinden.com'u Faz 4'e erteliyor (Cloudflare + zorunlu login duvarı, residential proxy ~$4/GB gerekir, $0 hedefiyle çelişir, hukuki risk) ve "yeniden açma sinyali" tanımlıyor. Lark ise sahibinden'i MVP hedefine koymuş. Crayon Camoufox/curl-cffi'yi anıyor ama "erken yapma" diyor.
Önerim: Katmanlı merdiven (Playwright son çare) + MVP pilot siteleri kolay olanlardan seç — plan-2'nin (Vatan + arabam.com) ve jellyfish'in (Vatan/İncehesap sınıfı JSON-LD'li siteler önce; Trendyol/HB ayrı bir spike) yaklaşımı doğru; lark'ın "en zor anti-bot'lu üçlüyle başla" planı MVP'yi silahlanma yarışına çevirir. Sahibinden → Faz 4 (jellyfish'in gerekçesi ikna edici). Crawlee'yi baştan alma (YAGNI) — sade fetch merdiveni + p-limit; browser-tier site sayısı artınca Crawlee'ye geç. Jellyfish'in OEM+PDF fikri "sıfır araba fiyat listesi" gerçekten istiyorsan değerli — bu bir kapsam kararı, sana kalmış.

6. Mobil kapsam ve kanallar
Mobil MVP içinde (3 plan): sketch (Sprint 5, Android-only — "iOS Windows'ta build edilemez, Xcode/macOS şart" ki bunu 4 plan bağımsızca doğruluyor), crayon (son MVP dilimi), plan-2 (en iddialısı: imzalı APK + ADB sideload, iOS simulator-only, iPhone için PWA kanalı).
Mobil MVP sonrası (5 plan): lark ("Telegram bot zaten telefonda — birincil mobil arayüz"), grove, jellyfish (Faz 3), plan-1 (spike sonrası), lamport (dashboard'dan hemen sonra Faz 5).
PWA: plan-2 birinci sınıf kanal (Apple ücreti ödemeden iPhone çözümü, /pair cihaz eşleştirme auth'u dahil); lamport "Tauri mobile tıkanırsa sigorta"; diğerleri değinmiyor.
Önerim: Lark'ın argümanı en güçlüsü: MVP'de mobil app yok, Telegram = mobil arayüz. Sıra: web dashboard → Tauri desktop → (MVP sonrası) Tauri Android. iOS'u "Later" kovasına koy (Windows gerçeği). iPhone'da dashboard'a bakmak istersen plan-2'nin PWA fikri en ucuz yol — ama ev sunucusu hosting'inde telefon erişimi için Tailscale benzeri bir çözüm gerekir; bu kararı hosting kararından sonra ver.

7. Süreç ağırlığı: tam Scrum mu, hafif Scrumban mı?
Tam Scrum simülasyonu (1 plan): sketch — ama en özgün katkısıyla: "No Fiction" kuralı (solo yapılamayan tören kurgulanmaz, kurgu doküman AI context'ini zehirler; her törenin gerçek-veriden üretilen solo karşılığı var) + apps/pm CLI (~400 satır: pm board, pm check CI'da doküman çürümesini yakalar) + YAML front-matter tek gerçek kaynak.
Scrumban/Kanban + Scrum sözlüğü (7 plan): lamport (fazlar, sprint yok), lark (Kanban+milestone+sezon gate'leri), grove (1 haftalık sprint, story point yerine S/M/L tişört bedeni), crayon (Kanban + terimleri öğren, opsiyonel haftalık ritim), jellyfish (Scrumban + spec-driven, WIP limiti 1, story point + velocity'yi "tahmin kalibrasyonu enterprise'da en çok sorulan beceri" diye tutuyor, ~60 satır board.ts + lefthook pre-commit ile BOARD.md üretimi), plan-1 (1-2 haftalık döngü), plan-2 (1 hafta, WIP 2).
Story point: tut (sketch, jellyfish — ölçüm şartıyla) vs bırak (lamport, grove).
Önerim: jellyfish'in Scrumban'ı + sketch'ten iki transfer. Taban: Kanban akışı + Epic/Story/Task hiyerarşisi + DoR/DoD + haftalık hafif ritim + retro. Sketch'ten al: (1) "No Fiction" kuralı — hangi artefaktı üretiyorsan gerçek veriden üret; (2) pm check fikrinin hafif hali (spec'siz task, DoD'suz done gibi tutarsızlıkları CI'da yakalayan küçük script). apps/pm'i tam ürün olarak yazmayı erteler, board.ts (~60 satır) ile başlarım — sketch'in CLI'ı eğlenceli dogfooding ama asıl ürünü geciktirir. Story point'i öğrenme hedefin için tut, gerçekleşenle karşılaştır (kalibrasyondan öğrenirsin), taahhüt baskısı yapma.

8. Branch stratejisi
main + dev + feature (6 plan): sketch, lark, grove, crayon, jellyfish (jellyfish'in notu: feat/* dalları şart çünkü PR diff'i blind review'ın girdisi), plan-2.
Sadece main + kısa ömürlü feature = trunk-based (2 plan): lamport ("tek geliştirici + CI'da dev'in katma değeri yok, senkron yükü var"), plan-1 ("uzun yaşayan dev branch olmayacak").
Önerim: trunk-based (main + kısa ömürlü feat/* + PR). Çoğunluğa rağmen azınlığın gerekçesi solo için daha doğru: dev branch'i tek kişide sadece merge bürokrasisi üretir. Jellyfish'in "PR diff'i review girdisi" mekanizması trunk-based'de de aynen çalışır. Solo'da bile PR aç (sketch'in de dediği gibi review disiplinini ve CI kapılarını çalıştıran mekanizma bu).

9. Review protokolü — yapı farkları
Herkes "biased + blind ikisi de değerli" diyor; ayrışma üç noktada:

Biased review'ı kim yapar? İki plan (lark, crayon) "implementer kendi işini self-review eder" diyor. Altı plan (sketch, jellyfish, grove, plan-1, plan-2, lamport) biased = spec'i bilen ayrı reviewer diyor; sketch bunu demir kural yapmış: "Implementer kendi işini aynı seansta review etmez — self-consistency bias en güçlü başarısızlık modu."
Kaç geçiş? sketch: 3 geçiş (A conformance / B blind-adversarial / C adjudication) + implementasyon öncesi ucuz "Spec Auditor" geçişi. plan-1/plan-2: blind + informed + reconciliation kaydı (plan-2 bulguları P0-P3 önceliklendiriyor). grove: 2 geçiş, "iki revizyon üstü = süreç tiyatrosu", küçük task'ta A+B tek prompt'ta birleşir. jellyfish: risk kademeli — yüksek risk (para/parser/kural motoru) = biased+blind+adversarial; orta = ikisi; düşük (UI kozmetiği/doc) = sadece biased.
Model çeşitliliği: sketch + grove blind reviewer için farklı model ailesi şart koşuyor (korele kör noktaları kırar).
Önerim: jellyfish'in risk-kademeli matrisi + sketch'in iki kuralı. Her story'ye DoR'da risk etiketi; düşük riskte tek review ile hız kaybetme. Kurallar: biased review'ı implementer değil ayrı seans yapar; blind reviewer'a spec sızdırılmaz ve mümkünse farklı model ailesi. Sketch'in "Spec Auditor" fikrini (implementasyondan önce design.md'yi ucuz modele denetletmek) yüksek riskli feature'larda ekle — en pahalı hatayı en ucuz anda yakalar. Çakışan bulguları jellyfish gibi review.md'nin "Anlaşmazlıklar" bölümüne yaz; bunlar senin mimari sezgini kalibre eden asıl malzeme.

10. Integration test veritabanı
Gerçek Postgres — Testcontainers/compose (4 plan): sketch, jellyfish ("in-memory SQLite ile Postgres'i taklit etmek, tam da bulmak istediğin SQL hatalarını gizler"), lark, plan-2.
PGlite in-process (1): grove. In-memory SQLite (1): lamport. Temp SQLite/Testcontainers karışık (1): crayon. (plan-1 SQLite kullandığı için tutarlı olarak SQLite ile test ediyor.)
Önerim: Testcontainers (veya docker-compose.test) ile gerçek Postgres. DB kararı Postgres olunca lamport'un in-memory SQLite'ı kendi içinde çelişkili kalıyor. PGlite hız istersen kabul edilebilir orta yol ama Testcontainers zaten yeterince hızlı ve CI'da (ubuntu runner'da Docker hazır) sorunsuz.

11. AI katmanı: SDK ve varsayılan sıra
Vercel AI SDK (2 plan): lamport (önerisi), lark (opsiyon olarak).
Kendi ince router'ın (6 plan): sketch (kilit gözlem: Gemini'nin OpenAI-uyumlu endpoint'i var, DeepSeek zaten OpenAI-uyumlu → tek adapter + config yeter), grove, crayon (Hermes'in fallback-chain tasarımını referans al ama bağımlılık yapma), jellyfish (+circuit breaker, +ai_calls maliyet tablosu, +aylık bütçe aşımında sert durdurma), plan-1, plan-2.
Varsayılan sıra felsefesi: ucuz-önce merdiven (lamport, crayon, grove, plan-2, sketch) vs görev-bazlı (lark: akıllı işe 3.7 Flash, basit işe Lite; plan-1: "ölçülmüş eval sonuçlarına göre seç") vs jellyfish'in uyarısı: DeepSeek V4 Flash çıkış tokeninde Flash-Lite'tan ~5.4× ucuz — senin istediğin sıra en ucuzu en sona koyuyor; sıra DB'de olduğu için varsayılanı yaz, ölç, değiştir.
Önerim: kendi router'ın (tek OpenAI-uyumlu adapter + config). 6/8 çoğunluk + öğrenme hedefi + gerçekten ince (~100 satır). Jellyfish'in üç eklentisi pazarlıksız alınmalı: çağrı başına maliyet logu, circuit breaker, bütçe sert-durdurma. Varsayılan sıra: senin istediğin (Lite → Flash → DeepSeek) ile başla — jellyfish'in dediği gibi bu koda gömülmüyor, bir hafta ölçüp DeepSeek'i öne almayı değerlendirirsin.

12. HTTP istemci zinciri ve Tauri CORS
openapi-typescript + openapi-fetch + openapi-react-query (1 plan): sketch.
Zod-first el yazımı wrapper (~40 satır) (7 plan): jellyfish'in karşı argümanı net: "bizde tek doğru kaynak Zod, OpenAPI ondan türetilecek — openapi-fetch ters yön."
Tauri CORS çözümü: sketch + jellyfish: @tauri-apps/plugin-http ile fetch'i değiştir (CORS tamamen devre dışı, sunucu ayarı gerekmez). lark: API'nin CORS allowlist'ine tauri://localhost ekle.
Önerim: Zod-first ince wrapper (timeout/AbortSignal, hata taksonomisi, Zod yanıt doğrulama) + Tauri'de plugin-http (iki en derin plan hemfikir; ortama göre transport seçen tek wrapper, uygulama kodu farkı bilmez). Sketch'in openapi zinciri güzel ama Zod-first zincirle çift kaynak yaratır.

13. Para saklama formatı
Integer minor unit (kuruş) + currency (3 plan): sketch ("float asla"), jellyfish (bigint), plan-1.
NUMERIC/decimal (2 plan): lark (numeric(12,2), "Drizzle string döndürür, servis katmanında çevir" tuzağını not ediyor), plan-2 (NUMERIC + API'de decimal string).
Diğer 3 plan belirtmemiş.
Önerim: integer minor unit (kuruş) + ISO currency kolonu. Float hatası sınıfını tamamen yok eder, Drizzle'ın numeric-string tuzağına hiç girmezsin, karşılaştırma/diff aritmetiği tam sayıyla trivial.

14. Snapshot yazma disiplini
Sadece değişimde yaz (2 plan, açıkça): sketch (snapshot_hash değişmediyse yalnız last_checked_at güncellenir — "tablo aksi halde günde binlerce gereksiz satır alır"), plan-2 (ilk gözlem + anlamsal değişiklikler + last_seen_at).
Her gözlemi yaz (append-only) (2 plan): lark (price_history her yakalamada + tracker'da denormalize last_price), jellyfish (observations append-only; raw_hash değişim tespitinde).
Diğerleri belirsiz.
Önerim: sketch kuralı — değişimde snapshot, değilse last_checked_at. Çalışma tarihçesi zaten ayrı scrape_runs tablosunda tutuluyor (birkaç planda ortak), veri kaybı yok. Grafik için "değişim noktaları" zaten yeterli ve daha temiz.

15. Auth iskeleti derinliği (MVP)
Sadece kolon + allowlist (çoğunluk): sketch ("kolon bedava, mantık pahalı" — user_id=1 sabit), jellyfish (owner_id + Telegram chat_id allowlist + statik API key + AuthGuard/CurrentUser soyutlaması — Faz 4'te sadece guard implementasyonu değişir), lamport, grove, lark (X-API-Key).
Gerçek tablolar Faz 0'da (1 plan): plan-1 (users + workspaces + memberships).
Cihaz eşleştirme sistemi MVP'de (1 plan): plan-2 (/pair <code> → HttpOnly session cookie, revocable device credential).
Önerim: jellyfish'in orta yolu — kolon + allowlist + statik API key + AuthGuard soyutlaması. Plan-1'in membership tabloları sınırda-fazla (workspace kolonu yeter, üyelik mantığı YAGNI). Plan-2'nin pairing'i zarif ama ancak dashboard internete açılırsa (PWA kanalı seçilirse) gerekir — o karara bağla.

16. Spec derinliği: ne zaman, kaç dosya?
Her story'ye spec: çoğunluk. Jellyfish önemli sınır çiziyor: spec'ler story seviyesinde üretilir — "tüm projeyi baştan spec'lemek waterfall'a döner, %60'ı çöpe gider."
Sadece büyük işlere spec (1 plan): plan-1 — "birkaç saatten büyük veya çok katmanlı işler; küçük bug = backlog maddesi + PR açıklaması."
Dosya sayısı: taban 3 (req/design/tasks — Kiro üçlüsü) → jellyfish 5 (+handoff.md +review.md) → plan-2 8 (+test-plan, implementation-brief, iki ayrı review dosyası).
Önerim: plan-1'in eşiği + jellyfish'in 5 dosyası. Küçük işler spec'siz akar (süreç tiyatrosu önlenir); spec alan işlerde handoff.md (implementer'ın "nerede saptım ve neden" notu) ve review.md sezonlar-arası akış için şart. Plan-2'nin 8 dosyası fazla — test planı design.md'nin bir bölümü olabilir.

17. Skills / AI context dosyaları
Kendi proje skill'lerini yaz (2 plan, en net): jellyfish — "model React'i zaten biliyor; bilmediği senin SourceAdapter arayüzün, senin test düzenin" → .claude/skills/add-source-adapter, write-story-spec, blind-review, new-migration. lamport aynı görüşte.
Doğrulanmış community skill'leri kur (2 plan): plan-1 (kurulum sayılarıyla: vercel-react-best-practices ~650K, nestjs-best-practices ~25.5K, tauri-v2 ~6.9K — "önce incele, körü körüne güvenme"), crayon (agents-inc/skills seçici).
AGENTS.md (cross-tool) + CLAUDE.md: crayon, plan-1, plan-2.
Önerim: ikisi birden ama öncelik sırayla. Önce CLAUDE.md + kendi 3-4 proje skill'in (jellyfish listesi birebir iyi) — en yüksek getiri burada. Sonra plan-1'in doğrulanmış listesinden yalnızca vercel-react-best-practices'i değerlendir. Birden çok AI aracı kullanacaksan kökte AGENTS.md (portable) tut.

18. MVP kapsam farkları (özellik seti)
Liste/kategori URL takibi (yeni ilan çıktı / ilan kalktı tespiti): sadece plan-2 MVP'ye koymuş (jellyfish arabam kayıtlı-aramayı "listing gibi modelle" diyerek dolaylı destekliyor). Diğerlerinde tekil ürün takibi var.
Kontrol aralığı: 15-30 dk (lamport) / 30 dk (lark) / 60 dk ürün-bazında ayarlanabilir (plan-1, plan-2) / 6 saat (jellyfish, ~20-50 ürün için) / saatlik (grove). Sketch sabit cron yerine rastgeleleştirilmiş aralık öneriyor (bot imzası bırakmamak için).
Bildirim hijyeni: dedupe_key + cooldown (jellyfish, plan-2, sketch) — aynı düşüş üç kez bildirilmesin.
Önerim: Tekil ürün takibiyle başla; liste/kategori takibini ikinci epic olarak al (araba tarafında asıl değeri o üretiyor — plan-2 haklı). Aralık: ürün-bazında ayarlanabilir, varsayılan 60 dk + sketch'in jitter'ı. Dedupe+cooldown pazarlıksız girsin.

C. Tek planda olup çalınmaya değer fikirler (hangi plan seçilirse seçilsin al)
sketch — "No Fiction" kuralı: kurgu artefakt üretme; her tören gerçek veriden üretilen çıktı verir. (AI context'ini yalandan korur.)
sketch — CONTEXT.md 200 satır sert limit: her seansın okuduğu invariant dosyası şişemez.
sketch — nightly live-smoke: adapter başına günde 1 gerçek URL; "alert sessizce sustu, haftalarca fark etmedim" felaketinin tek sigortası (jellyfish'in health_score + operatör alarmı fikriyle birleşir).
jellyfish — sanity guard'lar: fiyat ≤0 veya önceki gözleme göre >%70 sapma → "şüpheli" işaretle, bildirim gönderme, insan onayı iste. ("Yanlış fiyat, sessiz hatadan da kötü.")
jellyfish — ADR'ye "geri dönüş sinyali" alanı: "şu olursa bu kararı tekrar açarız" (plan-1'de de "yeniden değerlendirme tetikleyicisi" var). Vibe-coding'de karar körlüğüne panzehir.
jellyfish — AI bütçe sert-durdurma + ai_calls maliyet tablosu.
sketch — OpenAPI snapshot testi CI'da: FE/BE sözleşme sapması otomatik kırılır.
plan-2 — PWA + /pair cihaz auth: iPhone'a Apple ücreti ödemeden çıkış yolu (kanal kararına bağlı).
grove — MCP server fikri (Faz 2+): kendi API'ni ajanlara açmak; ilginç ama acele değil.
crayon — nuqs (URL state) ve DuckDB (post-MVP analitik): filtre/sıralama URL'de paylaşılabilir olsun; fiyat analitiği gerekirse pandas yerine SQL/DuckDB.
jellyfish — kanıtlı gate ritüeli: faz kapısı dosyasına her kriterin yanına komut çıktısı/SS/commit SHA yazılır — "kanıtla, iddia etme."
sketch — Walking Skeleton sprint'i: ilk sprint = uçtan uca en ince dilim ("bu sprint biterse proje gerçek") — jellyfish ve grove'un gate-0'ı da aynı ruhta; ilk hedef "bir fiyat DB'ye düşsün ve Telegram'a mesaj gelsin", ekran yok.
D. Ek soru 1 — Base olarak hangisi? Yoksa sıfırdan mı?
Sıfırdan yazmak daha kaliteli olmaz — net. Bu 8 planın içinde doğrulanmış araştırma gömülü (model isimleri/fiyatları, free-tier gerçekleri: Railway'in $1'a düşmesi, Fly'ın free tier'ı kaldırması, GH Actions 60-gün kuralı, Tauri-iOS-Windows imkânsızlığı, shadcn blok isimleri…). Sıfırdan yazan bir seans bunların bir kısmını yeniden türetemez veya yanlış hatırlar.

Tek planı olduğu gibi base almak da doğru değil, çünkü her biri çelişen "kilitli kararlar" içeriyor (madde 0). Yani cevabın ikinci kısmı doğru: base, senin kararlarından sonra netleşir — ama şimdiden söyleyebilirim:

Kalite sıralamam (derinlik + doğruluk + dürüstlük): 1) jellyfish (en derin araştırma, riskleri en dürüst yazan, scraping/maliyet gerçekçiliği), 2) sketch (en iyi süreç tasarımı + en keskin mimari düzeltmeler), 3) crayon (kütüphane güncelliği en iyi), 4) grove (en derli ADR tablosu), 5) plan-2 (kanal/kabul kriterleri en somut), 6) lark (en somut iş sıralaması ama Playwright-first zayıflığı), 7) plan-1 (ölçülü ve dürüst ama ince), 8) lamport (kompakt; Turso + in-memory-SQLite-test çelişkileri var).
Önerdiğim yol: Kararlarını verdikten sonra jellyfish'i iskelet al, sketch'in süreç bölümünü (No Fiction, review protokolü, CONTEXT.md) monte et, C bölümündeki tekil fikirleri ekle ve bunların hepsinden tek, taze bir konsolide plan (≤ ~350 satır) + ADR seti yazdır. Yani "sıfırdan yazım" değil, "kararlar sonrası sentez" — en kaliteli sonuç bu üçlünün ortası.
E. Ek soru 2 — Tek seferde inceleme problem mi? Doküman boyutu çok mu?
Tek seferde karşılaştırma: problem olmadı, işte yaptım. Toplam ~3.400 satır / kabaca ~100K token — bir karşılaştırma seansı için sınıra yakın ama tek geçişte işlenebilir. Bunu konu başına ayrı seanslara bölmek overengineering olurdu: her seans 8 dosyayı yeniden okuyup context'i yeniden kurardı, ve DB↔hosting↔scheduler↔bot gibi konular birbirine bağlı olduğu için parça parça bakınca çelişkiler görünmezdi.
Karar verme aşaması ayrı konu: 18 maddeyi tek nefeste cevaplamak yorucu; aşağıda kümeledim (F bölümü). Hepsini bu sohbette, küme küme cevaplayabilirsin — yeni seans açmak gerekmez.
Doküman boyutu: Karşılaştırma için kabul edilebilir; implementasyon context'i için kesinlikle çok. Zaten 8 planın kendisi de aynı şeyi söylüyor: implementer seansına asla "büyük plan" verilmez — sadece CLAUDE.md + CONTEXT.md + ilgili story spec'i verilir. Jellyfish tek başına 1073 satır — plan olarak bile kendi tavsiyesine göre bölünmeli. Nihai hedef: kararlar → tek konsolide plan → 8 dosya arşive.
F. Karar sıran (chat'te bu sırayla cevaplaman yeterli)
Cevap formatı serbest — "Küme A: hepsinde öneriyi kabul" ya da "1: Postgres, 2: Railway olsun, 3: ..." gibi kısa yazman yeterli, ben işlerim:

Küme A (birbirine bağlı, önce bunlar): 1-DB · 2-Hosting/scheduler · 3-Bot modu
Küme B (mekanik): 4-Monorepo · 8-Branch · 13-Para formatı · 14-Snapshot disiplini
Küme C (kapsam): 5-Scraping stratejisi + pilot siteler + sahibinden · 6-Mobil/PWA · 18-Liste takibi + aralık
Küme D (süreç): 7-Süreç ağırlığı · 9-Review protokolü · 16-Spec derinliği · 17-Skills
Küme E (teknik detay): 10-Test DB'si · 11-AI SDK + model sırası · 12-HTTP/CORS · 15-Auth derinliği
Küme F: C bölümündeki 12 tekil fikirden istemediğin var mı? (Varsayılanım: hepsi girer.)
Kararların geldikçe konsolide planı ve ADR taslaklarını buna göre çıkarırım.

Kararların işlendi. Cevabım dört bölüm: (1) sorularının cevapları, (2) 8. maddedeki önerinin değerlendirmesi, (3) atladığın/açık kalan kararlar, (4) sonraki adım. Hiçbir şeyi tool'la sormuyorum, hepsi burada.

1. Madde 2 — İkincil (fallback) scraping hattı: evet, yapılabilir ve ucuz
GitHub Actions zamanlı workflow'lar nasıl çalışır, sorunları ne?
Mekanik: Workflow dosyasına on: schedule: cron: '*/30 * * * *' yazarsın; GitHub bunu UTC saatiyle, paylaşımlı runner havuzunda çalıştırır. workflow_dispatch eklersen aynı workflow'u elle (veya API/gh CLI ile) de tetikleyebilirsin.

Üç bilinen sorun:

Gecikme/atlama (jitter): Cron "en erken şu saatte kuyruğa al" demektir, "tam şu saatte çalıştır" değil. Yoğun saatlerde başlangıç tipik olarak 3–10 dk, kötü günlerde 30 dk+ gecikir; nadiren o tetik tamamen atlanır. Garanti/SLA yok. Fiyat takibi için bu kabul edilebilir (sketch bile "fiyat takibi için tolere edilir" der), canlı bot komutları için edilemez.
60 gün kuralı ("sessizce kapanma"): Public repo'da 60 gün boyunca repo aktivitesi (commit) olmazsa zamanlı workflow'lar otomatik devre dışı bırakılır. Tam "sessiz" değil — GitHub önce uyarı e-postası atar ve Actions sekmesinde "disabled" bandı + tek tıkla yeniden etkinleştirme butonu çıkar — ama e-postayı kaçırırsan pratikte sessizdir. Geliştirme sürerken sorun değil (commit'ler sayacı sıfırlar); proje "bitti ve kendi kendine çalışıyor" moduna geçince risk başlar. Ayrıca cron'u en son değiştiren kullanıcı repo erişimini kaybederse de workflow durur (solo'da alakasız).
Datacenter IP: Runner'lar Azure IP'leridir → Cloudflare/DataDome korumalı sitelerde blok oranı ev IP'sinden belirgin yüksek. Yani bu hat kolay siteler için (JSON-LD'li Vatan sınıfı) işe yarar, zor siteler için zaten çözüm değil.
Önerdiğim ikincil hat tasarımı (koordinasyon dahil)
Neon'u seçmenin bonusu tam burada: DB buluttan her yerden erişilebilir, iki hat da aynı DB'ye yazabilir.

Primary: ev sunucusu, in-process scheduler, her başarılı koşuyu scrape_runs'a lane='primary' ile yazar.
Secondary: aynı scraper CLI'ı (apps/worker'ın CLI entry'si) GH Actions'ta 60 dk cron'la koşar, SCRAPE_LANE=backup env'iyle. İlk işi DB'ye bakmak: son primary koşusu < 3 saat önceyse hiçbir şey yapmadan çıkar (no-op, ~10 saniye runner süresi). Primary bayatsa scrape eder, Neon'a yazar, Telegram'a bildirir (bot token GH secret olarak). Böylece çifte scraping, çifte bildirim, config değiştirme derdi yok — hatlar DB üzerinden kendi kendine koordine olur.
Dead-man's switch (iki hattın da ölmesine karşı): healthchecks.io (ücretsiz) — primary her başarılı döngüde bir ping URL'ine dokunur; X saat ping gelmezse healthchecks sana e-posta/Telegram ile "sistem sustu" der. "Alert gelmiyor çünkü her şey sakin mi, yoksa sistem mi öldü?" belirsizliğinin tek gerçek çözümü bu. (60-gün kuralına karşı da sigorta: Actions kapansa bile haberin olur.)
Cloudflare Workers webhook gerekecek mi? — Hayır, şimdi değil.
Kritik teknik gerçek: Telegram'da bir bot ya long-polling ya webhook modundadır, ikisi aynı anda olmaz (setWebhook çağırınca getUpdates kapanır). Yani CF Workers'ı "yedek komut alıcısı" olarak primary'nin yanına koyamazsın. İkincil hat zaten yalnızca scrape + bildirim işini kurtarıyor (bunlar için webhook gerekmez — Actions job'ı sendMessage API'sini doğrudan çağırır). Ev sunucusu kapalıyken bot komutları cevapsız kalır — bu bilinçli kabul edilen risk. Botu 7/24 yapmak istersen çözüm "yedek" değil taşımadır: bot tamamen CF Workers webhook'a geçer (grove'un Faz 2 planı). Bunu ADR'ye "geri dönüş sinyali" olarak yazarız: "ev sunucusu uptime'ı aylık %95'in altına düşerse veya komutlara dışarıdan erişim ihtiyacı doğarsa bot CF Workers webhook'a taşınır."

2. Madde 4 — Tasarım pattern'leri görüşüm (onion, DDD, CQRS, mediator, vertical slice, ES, saga)
Genel ilke: pattern ihtiyaca cevaptır; CV için pattern eklemek anti-pattern'dir. Bu projede mülakatta anlatacağın güçlü cümle zaten şu: "Nx module-boundary lint'iyle zorlanan, ports & adapters'lı modüler monolit."

Hexagonal / Ports & Adapters (onion'un pratik hali) → EVET, zaten planın omurgası. SourceAdapter, AiProvider, Notifier, JobQueue portları tam bu. Onion/clean/hexagonal aynı ailenin jargon varyantları; kural tek: bağımlılıklar içe akar, libs/domain saf kalır, hiçbir şeye bağımlı olamaz. Nx boundary kuralı bunu mekanik olarak polisler — kağıt üstünde değil.
DDD → lite evet, tören hayır. Alacaklarımız: ubiquitous language (glossary zaten var), value object olarak Money (minor unit kararınla doğal eşleşiyor), repository pattern. Almayacaklarımız: aggregate root törenselliği, domain event bus, Unit of Work (MikroORM'u tüm planlar aynı gerekçeyle reddetti). Stratejik DDD'nin "bounded context"leri bizde gayriresmî olarak Nest modülleri: scraping / tracking / notify / ai.
CQRS → hayır (framework olarak). @nestjs/cqrs + command/query bus solo CRUD-ağırlıklı app'te saf boilerplate. Fikrin özü zaten hafifçe var: yazma yolu (snapshot/diff) ile okuma yolu (dashboard sorguları) SQL seviyesinde ayrı. Formal CQRS ancak event sourcing gelirse anlamlanır — gelmiyor.
Mediator (MediatR tarzı) → hayır. Mediator, .NET dünyasında controller şişmesine çözümdür; NestJS'te DI + modül sistemi o işi zaten görür. CommandBus = mediator'dur, yani 3'le aynı ret.
Vertical slice → süreçte evet, kod düzeninde karma. Her story zaten dikey dilim (UI→API→DB uçtan uca) — bu süreç kararın verildi. Kod düzeninde pragmatik karışım: apps/* içinde feature-bazlı modüller (dikey), libs/* (contracts/db/domain) yatay paylaşım katmanları. Nx tam bu karışımı teşvik eder; çelişki değil.
Event sourcing → hayır. Tüm planlar YAGNI listesine koymuş, katılıyorum. Ama şu tespit hoş: observations/price_snapshots zaten append-only bir olay günlüğü — tarihçenin gerçekten önemli olduğu tek yerde "fakirin event log'u" bedavaya var. State'i event'lerden yeniden kurma makinesi (gerçek ES) maliyet, sıfır getiri.
Saga / process manager → hayır. Saga dağıtık, uzun yaşayan işlemler içindir; bizde tek DB + tek monolit var. En yakın ihtiyacımız scrape→detect→notify hattı — düz sıralı kod + idempotency (dedupe_key) çözer.
Bonus, almaya değer minik şey — Outbox-lite: bildirimi önce DB'ye pending yaz, ayrı dispatcher sent'e çevirsin (lark ve plan-2'de zaten var). Crash anında bildirim kaybolmaz. Tam outbox/CDC değil, tablosu kadarı yeter.
Özet: al = hexagonal portlar, repository, Money value object, outbox-lite; alma = CQRS, mediator, ES, saga, aggregate töreni. CQRS/ES/saga'yı öğrenmek istiyorsan ayrı mini projede öğren — bu ürünü laboratuvar yapma.

3. Madde 6 — Full cross-platform + PWA: çakışmaz, tek koşulla
Aynı React codebase'in 5 çıktısı olur, birbirini bozmaz (plan-2 modeli):

Web SPA (dashboard, localhost) — MVP
PWA = aynı build + manifest + service worker (vite-plugin-pwa; service worker sadece web build'de aktif, Tauri build'lerinde kapalı — bilinen tek teknik çakışma noktası bu ve çözümü bu koşullu kayıt)
Tauri desktop (Windows lokalden; macOS/Linux CI'dan)
Tauri Android — imzalı APK, ADB ile telefona; mağaza yok
Tauri iOS — build + simulator smoke, ücretli üyelik yok, gerçek iPhone'a native kurulum yok → gerçek iPhone kanalın = PWA
İki bağımlı gerçek:

PWA'nın API erişimi: Ev sunucusu + long-polling kararında public URL yok; telefondaki PWA evin dışından API'ye ulaşamaz. Çözüm önerim: Tailscale (ücretsiz) — telefon + PC aynı tailnet'te, Tailscale Serve ile HTTPS de bedava (iOS PWA kurulumu HTTPS ister), port açma/DDNS yok, dışarıya hiçbir şey açılmaz → auth olarak allowlist+API key yeter, plan-2'nin pairing sistemi gerekmez. Alternatif: Cloudflare Tunnel (public URL verir; ileride başkasıyla paylaşacaksan bu + pairing). Kararın: Tailscale mi, CF Tunnel mi? (Önerim Tailscale.)
macOS/iOS build'leri CI'da macOS runner ister (Windows'ta ikisi de derlenemez — desktop macOS dahil). Private repo'da macOS dakikaları 10× sayılır; public repo'da standart runner'lar (macOS dahil) tamamen ücretsiz. Yani "full cross platform + $0" hedefi repo görünürlüğü kararına bağlanıyor → aşağıda madde 5-B.
4. Madde 8 — "main + dev, PR yok, feat yok" değerlendirmesi: evet, mantıklı — iki ayarla
Önce dürüst teyit: önceki turda trunk-based + PR önermiştim; senin kurgun kendi review kararınla (madde 9: faz geçişinde iki review promptu) birleşince aynı işi daha az törenle görüyor, çünkü blind review'ın ihtiyacı olan diff zaten main..dev'de birikiyor: dev'de çalışırsın, faz sonunda git diff main..dev = review girdisi, review geçince main'e merge = faz kapısı. Öneriyi geri çekiyorum, senin modelin kabul. İyi çalışması için iki ayar:

CI iki kademeli olsun: dev push'larında hızlı doğrulama (lint + typecheck + unit/integration) — yoksa hataları faz sonunda topluca bulursun ve review diff'i çöple dolar; main merge'inde ağır kademe (e2e + build artifact + release işleri). "main = CI/CD tetikleyici" fikrinin rafine hali bu.
Commit disiplini şart: branch izolasyonu olmadığından tek izleme aracın tarihçe. Conventional commit + task ID (feat(scraper): vatan adapter (PT-014)) ve "her commit çalışır durumda" kuralı. Faz diff'i çok büyürse review promptuna commit aralığı vererek dilimlersin.
Kabul edilen küçük riskler (bilerek): faz ortasında dev kırılırsa dönüş noktası bulmak biraz zahmetli (çözüm: ara tag), ve birden çok iş aynı diff'te karışır (çözüm: yukarıdaki commit disiplini). İkisi de solo için katlanılır.

5. Atladığın / hâlâ açık kalan kararlar (asıl sorunun cevabı)
Büyük resimde her şey kapandı; şu 5 küçük karar açık:

A. Dokümantasyon dili — TR mi EN mi? Kod/identifier zaten EN olacak. Docs için önerim: EN (ADR/spec/runbook) — kariyer/portfolio hedefinde mülakatta açıp göstereceğin arşiv bu; retro/daily gibi kişisel notlar TR kalabilir. (UI dili ayrı konu: TR, i18n yok — konsensüstü.)
B. Repo görünürlüğü — public mi private mı? Önerim: public. Gerekçe: Actions tamamen ücretsiz olur (macOS runner dahil → iOS/macOS build'lerin bedava, madde 6'nın ön koşulu), portfolio değeri, secrets zaten GH Secrets'ta. Karşı argüman istersen: kod herkese açık olur.
C. Sıfır otomobil (OEM marka siteleri + PDF fiyat listeleri) kapsamı. Jellyfish'e özgü fikirdi, açık bırakmıştım. Önerim: MVP dışı, Faz 2 epic'i — MVP pilotları notebook (Vatan) + ikinci el (arabam.com) olarak kalsın. İstersen MVP'ye alırız ama unpdf + tablo çıkarımı ayrı bir iş yükü.
D. PWA erişim yolu: Tailscale mi Cloudflare Tunnel mi? (Madde 3'te anlattım; önerim Tailscale, pairing'siz.)
E. (Onay yeterli) Madde 2'deki ikincil hat tasarımı: "staleness-check'li Actions backup lane + healthchecks.io dead-man's switch, CF Workers yok" — bu şekilde planlıyorum, itirazın varsa söyle.
Ayrıca kayda geçirdiğim (karar gerektirmeyen) notlar: pnpm ✓ (zaten tüm planların varsayımıydı, Nx+pnpm), Kiro IDE ile spec üretimi ✓ — konsolide plana .kiro/specs + .kiro/steering uyumu ve docs/prompts/ altında 5 hazır prompt (planner/spec, implementer, informed review, "git changes review" [blind], handoff şablonu) yazılacak; skills konusu minor-todo olarak backlog'a ✓; atlanan milestone'lara sebep yazılmış kısa doküman ✓ (No-Fiction kuralının doğal uzantısı); tam yol haritası (MVP→Post-MVP→SaaS→Enterprise) plana eksiksiz girecek ✓; task ID prefix'i olarak PT-### kullanacağım (itiraz yoksa).

6. Sonraki adım
A–E'yi cevapladığın an (kısaca "A: EN, B: public, C: Faz 2, D: Tailscale, E: onay" gibi yazman yeter — ya da tek satır "hepsinde önerin"), konsolide planı (≤~350 satır, jellyfish iskeleti + sketch süreç kiti + C fikirleri + bütün kararların) ve ADR setini yazıp repo'ya (docs/ iskeletiyle birlikte) çıkaracağım.

GPT 5.6 Sol Max çalışmasını satır satır işledim, kendi önerilerimle çarpıştırdım ve b-2/b-3 talimatın gereği tüm kararları+notları+alternatifleri kalıcı bir karar defteri dosyasına yazıyorum (bundan sonra konsolide plan ve ADR'ler bu defterden türeyecek). Önce dosyayı oluşturuyorum, sonra a-1/a-2 raporu aşağıda.

Write [repo]\docs\decision-ledger.md
84 lines
# Decision Ledger — AI Price Tracker

> Single source of record for every planning decision (2026-08-21/22), including rejected
> alternatives, objections raised by independent AI reviewers (8 plans + GPT 5.6 Sol review),
> and reopen triggers. The canonical plan and the wave-1 ADRs are derived from this file.
> Decisions not promoted to a full ADR live here until they are touched again ("promote on touch").
>
> Sources compared: recursive-lamport (RL), tender-sketch (TS), linear-lark (LL),
> federated-grove (FG), replicated-crayon (RC), luminous-jellyfish (LJ), plan-1 (P1),
> plan-2 (P2), GPT 5.6 Sol Max cross-review (GPT), Claude Fable 5 synthesis (CF).
> Owner decisions by GreyHappi are final.

---

## 1. Locked decisions

| ID | Decision | Key rationale / rejected alternatives | Reopen trigger |
|---|---|---|---|
| D-01 | **PostgreSQL from day 1** — Neon free tier (runtime) + Docker Postgres (local dev & integration tests). Helper tools allowed: Neon CLI/MCP, psql, Docker Desktop. | Drizzle schemas are dialect-specific (`sqliteTable` ≠ `pgTable`) → "migrate later" is a real rewrite; SQLite single-writer lock (scraper+API+bot write concurrently); type loss (timestamptz/jsonb/partial idx). Rejected: SQLite/Turso start (RL, P1), dual-dialect config (RC). | Neon free limits (0.5 GB / 100 CU-h) exceeded → self-hosted PG or paid tier |
| D-02 | **Runtime: home server primary (Docker) + GitHub Actions backup lane.** Primary: in-process scheduler, logs every run to `scrape_runs` with `lane='primary'`. Secondary: same worker CLI on 60-min Actions cron with `SCRAPE_LANE=backup`; first step checks DB — if last primary run < 3 h ago → no-op exit (~10 s runner time); if stale → scrape, write to Neon, notify via Telegram sendMessage (bot token in GH secrets). Lanes self-coordinate through DB timestamps — no double scraping/notification. **Dead-man's switch: healthchecks.io** — primary pings on every successful cycle; missing pings for X h → alert. Covers "system silent vs. nothing changed" ambiguity AND the Actions 60-day auto-disable. | Residential IP is a real scraping advantage (datacenter IPs blocked far more aggressively — TS, LJ); $0. Rejected as primary: GH Actions cron (queue jitter 3–30 min, occasional dropped runs, public-repo 60-day inactivity auto-disable, DC IP), Railway free ($1/mo credit — effectively dead), Render free (cold start), Fly (no free tier). TS's claim "Actions cannot be a scheduler" was **too absolute** (GPT correction): it is a legitimate best-effort scheduler, not an SLA one — hence backup lane, not primary. | Monthly uptime of primary < ~95% or need for 24/7 → re-architect (see D-03 trigger) |
| D-03 | **Telegram bot: grammY, long-polling on primary.** No port forwarding/DDNS/reverse proxy/TLS needed. | **A bot is either polling or webhook, never both** (`setWebhook` disables `getUpdates`) → CF Workers cannot be a *standby* command receiver. Backup lane covers scrape+notify only (plain `sendMessage`, no webhook needed). Commands unavailable while primary is down = **accepted risk**. | If home-server uptime < 95%/mo or external command access needed → **migrate** (not "back up") bot to Cloudflare Workers webhook (FG Phase-2 design) |
| D-04 | **Monorepo: Nx + pnpm**, `enforce-module-boundaries` from day 1. **Lean layout:** `apps/{api,worker,web}` (+`native` when the Tauri phase starts), `packages/{contracts,domain,db,scraping,ai,notifications,testing}`. Bot = module inside `api`; worker = thin entrypoint over shared services. No empty scaffold apps (landing/pm/mobile). | 7/8 plans chose Nx; boundaries are the only automatic guard against architecture rot; career-learning goal. Rejected: Turborepo (FG), npm (pnpm chosen). | — |
| D-05 | **Patterns: modular monolith + hexagonal ports** (`SourceAdapter`, `AiProvider`, `Notifier`, job-trigger seam), repository pattern, `Money` value object, **outbox-lite** for notifications (write `pending` row → dispatcher marks `sent`; crash-safe). DDD-lite only (ubiquitous language/glossary, value objects). | Rejected: CQRS framework (`@nestjs/cqrs` = ceremony solo), MediatR-style mediator (Nest DI already does it), event sourcing, saga (no distributed tx), aggregate ceremony, full JobQueue framework now (GPT: `run-due-checks` idempotent command is enough — see D-26). **Note (kept deliberately):** `observations` is an append-only "poor man's event log" — history exists exactly where it matters, at zero cost; a real ES rebuild machine would be cost with zero return. | CQRS/ES/saga only if a genuinely distributed workload appears (none foreseen) |
| D-06 | **Scraping: layered ladder, Playwright last.** 0) official API/feed if any → 1) native fetch → 2) JSON-LD / meta / embedded state (`__NEXT_DATA__`) → 3) site adapter + Cheerio → 4) Playwright only when forced. PDF parsing (unpdf) for OEM price lists. No CAPTCHA/login/anti-bot bypass — such sources are marked **unsupported**. Crawlee deferred (add when category crawling/session pools are real). **AI extraction is quarantined:** last-rung output enters as `extraction_method='ai'` with confidence, flagged suspicious, never fires an alert without human confirmation; also used to draft parser repairs (human-approved). | Rejected: Playwright-first (LL — target-site choice made it seem necessary), Crawlee day 1 (TS, RC), stealth/proxy stack now (RC). Sanity guards (LJ): price ≤ 0 or >70% swing → suspicious, no alert. | Crawlee: when list crawling ships (Phase 2). Proxy budget: sahibinden reopen only |
| D-07 | **MVP scope (owner decision):** three categories, sequenced strictly — 1) notebook (Vatan class, JSON-LD), 2) used-car listing (arabam.com), 3) **new-car OEM price lists (≥1 brand, PDF-capable) — pulled INTO MVP by owner** against GPT's narrower advice; mitigation = sequencing + walking-skeleton gate first. Direct product/listing URL tracking only; **category/list-URL tracking = Phase 2 epic** (new/removed listing detection is a distinct product problem). Default check interval 60 min + jitter, per-target override. sahibinden.com → Phase 4 (Cloudflare + login wall + ToS risk; reopen: ~$10/mo proxy budget AND 1 month stable MVP). | GPT objection recorded: "max two pilot adapters; OEM later". Owner overruled; risk managed by ordering — nothing starts before the first real notification works end-to-end. | — |
| D-08 | **Platforms: full cross-platform Tauri 2.** MVP: Telegram + responsive web dashboard + **PWA** (same React build; service worker registered only in the web profile via vite-plugin-pwa, disabled in Tauri builds). Then: Tauri desktop → **Android signed APK** (personal keystore, ADB sideload, no store) → **iOS build + simulator smoke only** (no paid Apple account; real iPhone channel = PWA). macOS/iOS/desktop matrix built on GH Actions macOS/ubuntu/windows runners — free because repo is public (D-24). | Rejected: Flutter (2nd codebase/language), store distribution now, paid Apple membership. Windows cannot build macOS/iOS locally — CI does it. | TestFlight/store only if payment accepted later |
| D-09 | **Remote access: Tailscale** (+ Tailscale Serve for HTTPS — iOS PWA install requires secure context). Nothing exposed to the public internet; no port forwarding. | Rejected: Cloudflare Tunnel (public URL — revisit if sharing with family/others; would then require D-10 pairing), LAN-only (too weak). | Sharing with other people → CF Tunnel + P2's pairing/session design |
| D-10 | **Auth (MVP): `workspaces` table + single seed workspace; `workspace_id` on aggregate roots only** (children reach it via parent — GPT refinement over TS's "every table"); Telegram `chat_id` allowlist; static API key behind an `AuthGuard` abstraction (guard implementation swaps in SaaS phase, controllers unchanged). Static keys never shipped into a public browser context (dashboard is Tailscale-only). | Rejected: membership/OAuth/RBAC/billing now (P1's Phase-0 membership tables = slightly too much), P2 pairing system (only needed if public exposure — see D-09 trigger), no columns at all (RL). | Public exposure → pairing/real auth mandatory |
| D-11 | **Money = bigint minor units (kuruş) + ISO-4217 currency column; API serializes as string. Float never.** All time = UTC `timestamptz`; localization at presentation. | Rejected: NUMERIC(12,2) (LL, P2 — valid but minor-unit arithmetic is deterministic and dodges Drizzle's numeric-string trap). | — |
| D-12 | **History discipline: write `observations` only on semantic change** (`snapshot_hash`), otherwise update `last_checked_at`. `scrape_runs`/`tracking_runs` records **every attempt** (lane, status, duration, error class). `change_events` = what changed; `notification_deliveries` = what was sent, with `dedupe_key` + cooldown (same drop never notified twice). **Raw HTML on failure: primary → local disk (7-day rotation); backup lane → Actions artifact; never into Neon** (0.5 GB free cap — GPT sizing guard). | Rejected: append-every-scrape history (LL; storage noise), storing raw HTML in DB. | — |
| D-13 | **Data model: two-level `tracking_target` (logical item) → `source_entries` (per-site URL/listing).** No automatic cross-site product matching — grouping without human approval never happens; a heavier catalog/product-matching layer is deferred. (GPT warning honored: URL is not a canonical product; matching is a hard problem — but the two-level schema itself is cheap and keeps cross-site comparison possible later. Open: O-02.) | Rejected: flat product+history only (RL/LL), auto-matching. | O-02 answer may add a comparison UI epic |
| D-14 | **API: single `/api/v1`** (NestJS URI versioning) for web/PWA/desktop/mobile/bot. OpenAPI **derived from Zod** (`packages/contracts` is the single source of truth); **OpenAPI snapshot test in CI** (contract drift breaks the build). | Rejected: BFF split (unanimous), openapi-fetch codegen chain (TS — reverse direction vs Zod-first). | — |
| D-15 | **HTTP client: Zod-first typed fetch wrapper** (~40 lines: timeout/AbortSignal, error taxonomy, Zod response validation, auth header). **Tauri builds swap transport to `@tauri-apps/plugin-http`** (bypasses CORS entirely); app code is transport-agnostic. | Rejected: axios (unanimous), ky (wrapper suffices), CORS allowlist approach (LL — plugin-http cleaner, no server config). | — |
| D-16 | **AI: own router** — single OpenAI-compatible adapter + config (Gemini and DeepSeek both speak the OpenAI protocol). **Two real providers wired, third is config** (GPT). **Task-based routes over a default chain** (GPT refinement, adopted light): normalize = cheap; candidate ranking = cheap, escalate on low confidence; parser repair = strong + human approval; default chain `gemini flash-lite → gemini flash → deepseek v4 flash` — order lives in DB, editable via `/model` + dashboard; model IDs re-verified at integration time (they are config, never code). Circuit breaker; every call logged to `ai_calls` (tokens, cost, latency, purpose); **monthly hard budget stop + Telegram alarm**. Build a small **eval set (20–50 real samples)** before locking the order (P1/P2/GPT). | Rejected: Vercel AI SDK (RL — extra dep, learning goal favors own ~100-line router), Hermes/OpenClaw as runtime deps (unanimous; Hermes fallback-chain kept as design reference). Note: LJ observed DeepSeek is ~5.4× cheaper on output than Flash-Lite — measure, then reorder. Naming: "GPT-5.6 Luna Max" is not a model id — id is `gpt-5.6-luna`, `max` is a reasoning-effort setting (GPT correction). | Reorder after eval-set results |
| D-17 | **Testing:** Vitest everywhere (FE+BE); **Testcontainers real Postgres** for backend integration (in-memory SQLite hides exactly the SQL bugs you want to find); RTL + MSW for frontend integration; 3–5 critical Playwright E2E to start; **fixture contract tests are regression tests only — they do NOT detect live site changes** (GPT semantics correction); the pair fixture + **nightly live-smoke health job** (non-CI-blocking, 1 real URL per adapter, failure → Telegram) is the actual defense; fast-check property tests targeted (price parsing, threshold math, dedupe/idempotency); **coverage gates only on parser/domain packages — no repo-wide % gate** (metric gaming risk, GPT). Tauri native smoke (tauri-driver) at the native phase. | Rejected: PGlite (FG), in-memory SQLite (RL), global 80% gate (LJ had domain≥80 — narrowed). | — |
| D-18 | **Process: Scrumban.** 1-week cycles, **WIP = 1**, sprint goal + demo/review + short retro; no daily standup. **No-Fiction rule (TS): every artifact is generated from real data — no fictional ceremonies.** Skipped milestones/ceremonies get a short doc stating **why** they were skipped (owner addition). **Story points kept lightweight** (owner) **+ actual cycle time recorded from day 1** (GPT addition) — calibration reviewed after ~3 cycles. | Rejected: full Scrum simulation (TS — theater risk solo), dropping SP entirely (GPT/FG). | — |
| D-19 | **Reviews: at phase transitions, not per story** (owner simplification). Two AI prompts, each in a fresh season: (1) **informed/conformance review** — sees requirements + design + handoff + diff; (2) **independent "git changes review"** — sees neutral goal + **acceptance criteria** + diff + test results, but NOT design rationale/handoff/chat history (GPT/P1/P2 refinement: pure-diff reviewers can't know intended behavior). High-risk work (money, dedupe, auth, migrations, AI budget) adds an **adversarial pass**. Findings merged in a single reconciliation record. Diff source = `main..dev`. Implementer self-check = preflight, never counts as review. | Rejected: per-story dual review (cost), implementer-as-biased-reviewer (LL/RC — self-consistency bias). | Collaboration starts → per-PR reviews |
| D-20 | **Git: `main` + `dev` only. No PRs, no feature branches** (owner decision, GPT objected — recorded). Work happens on `dev` with conventional commits + task id (`feat(scraper): vatan adapter (PT-014)`), every commit working; mid-phase tags as restore points. **CI two-tier:** `dev` push → lint/typecheck/unit/integration; `main` merge → e2e/build/release. `main..dev` diff doubles as the phase-review input. | Rejected: PR + short-lived feat/* flow (CF's original & GPT's rec — solo + phase-gate reviews make it redundant ceremony). | A collaborator or a real staging env appears → switch to PR flow |
| D-21 | **Specs: Kiro IDE-generated** (`.kiro/specs/<id>/requirements.md` EARS + `design.md` + `tasks.md`) **only for work above a few hours or spanning layers** (P1 threshold); small work = board entry + commit trail. Season outputs live alongside: `handoff.md` (implementer: what I did, where I deviated and why) + `review.md`. Test plan is a section of `design.md` (not a separate file — P2's 8-file spec rejected). **Prompt kit in `docs/prompts/`:** planner/spec prompt, implementer prompt, informed-review prompt, git-changes-review prompt, handoff template. | Rejected: spec-everything (waterfall drift), 8-file specs (P2). | — |
| D-22 | **Docs: English** (career/portfolio artifact); personal notes (retro/daily) may stay Turkish. **AGENTS.md = canonical repo contract; CLAUDE.md = tool-specific deltas only, never a copy** (GPT; fits the multi-tool reality: Kiro + Claude + GPT). Lean tree: `docs/{product,architecture,decisions,roadmap.md,work/board.md,specs,runbooks,prompts}`. `work/board.md` generated by ~60-line `board.ts` from story frontmatter + lefthook pre-commit (GPT alternative "manual until drift" recorded; generator kept — trivial cost, doc-rot is the #1 AI-workflow killer). **Canonical plan target: 200–350 lines**; details pushed to ADR/spec/runbook. **ADRs in two waves: ~7 core ADRs before coding** (DB, runtime+lanes, ORM, monorepo, bot mode, scraping strategy, money/time), everything else stays in this ledger and is promoted on touch (GPT's "don't write 18 ADRs before code" — adopted). | Rejected: 18 upfront ADRs (LJ), giant docs trees day 1 (TS/LJ/P2 full breadth), CLAUDE.md-as-canonical. | — |
| D-23 | **i18n: IN MVP (owner override of the unanimous YAGNI advice** of all 8 plans + GPT + CF): UI in **EN/TR/AR**. Arabic ⇒ **RTL from day 1**: logical properties (`ms-/me-/ps-/pe-`, start/end), `dir`-aware components (Radix DirectionProvider), rtl-safe reviews part of DoD for UI stories. Framework: react-i18next (mainstream, best AI training coverage). `locale` column on users/workspace from day 1. Estimated +15–25% UI effort — accepted by owner. Open: O-01 (bot/notification languages). | Objection recorded: every source called MVP i18n YAGNI; owner wants it — building RTL-safe from day 1 is far cheaper than retrofitting, which is the honest mitigation. | — |
| D-24 | **Repo: public on GitHub.** Unlimited free Actions on standard runners **including macOS** (enables D-08 build matrix at $0); portfolio value. Secrets in GH Secrets; keystore/passwords never committed. | Rejected: private (2000 min/mo cap, 10× macOS multiplier). | — |
| D-25 | **Skills: minor backlog todo.** Start with AGENTS.md/CLAUDE.md only; own project skills (add-source-adapter, write-story-spec, blind-review, new-migration) later; community skills (e.g. vercel-react-best-practices) evaluated when touched. | — | — |
| D-26 | **`run-due-checks` idempotent worker command (GPT, adopted):** one command = scrape → compare → persist → notify, safe to run twice. The **single scheduling surface**: home scheduler calls it, the Actions backup lane calls it, manual Telegram `/check` calls it. No generic JobQueue framework needed for this. | Supersedes "JobQueue port day 1" from TS/LJ — the seam is this command; a queue interface appears only when a queue appears. | Multi-worker need → introduce queue behind the same command |
| D-27 | **Canonical plan = fresh synthesis** (owner-approved): written new at P1-like editorial density (200–350 lines), content harvested from LJ (technical depth: scraping ladder, risk thresholds, cost realism) + TS (process kit: No-Fiction, review protocol, CONTEXT discipline) + this ledger. **No plan adopted verbatim.** Also adopted from TS: `CONTEXT.md` with a hard 200-line cap (the file every AI season reads first); OpenAPI snapshot gate; nightly live smoke. **Sequence: 1–2 day risk spike on the pilot sources BEFORE foundation** (GPT) → minimal foundation → walking skeleton (one URL → extract → PG → change → Telegram; "no screens, no beauty — a price lands in the DB and a message arrives"). | Base-plan debate resolved: GPT preferred P1-as-base, CF preferred LJ+TS-as-quarry — both converge on fresh synthesis; editorial style from P1, substance from LJ/TS. | — |

---

## 2. Recorded notes, objections & corrections (mention in canonical plan where relevant)

- **N-01 — CF Workers webhook analysis (full reasoning preserved):** Telegram bots are polling XOR webhook (`setWebhook` kills `getUpdates`), so a serverless webhook cannot sit beside the polling primary as a "standby command receiver". The backup lane only rescues scrape+notify (plain `sendMessage`, no webhook needed). Commands going silent while the primary is down is a consciously accepted risk. Going 24/7 for commands is a **migration**, not a backup: move the bot wholly to CF Workers webhook (FG Phase-2 design). Reopen trigger lives on D-03.
- **N-02 — GPT 5.6 Sol divergences and outcomes:** base plan (→ D-27 synthesis), git flow (owner kept main+dev; GPT's PR flow = reopen trigger on D-20), MVP breadth incl. OEM (owner overruled; sequencing mitigation, D-07), i18n in MVP (owner overruled all reviewers, D-23), story points (kept + GPT's actuals, D-18), board generator (kept vs GPT's manual-until-drift, D-22), ADR count (GPT won: 7 core + ledger, D-22), AGENTS.md canonical (GPT won, D-22), coverage gates (GPT won: targeted only, D-17), blind-review inputs (GPT/P1/P2 refinement adopted, D-19), workspace_id placement (GPT won: roots only, D-10), task-based AI routing (adopted light, D-16), AI-extraction quarantine (adopted, D-06).
- **N-03 — Fixture test semantics (correction adopted):** a committed HTML fixture test catches *code regressions against known HTML*; it can NOT detect that the live site changed (the fixture is frozen). Live-site drift is caught only by the scheduled live-smoke/health job. TS/LJ overloaded the fixture tests' meaning; wording fixed in D-17.
- **N-04 — "Actions cannot be a scheduler" (TS) softened:** GitHub schedules workflows fine but best-effort — queue jitter (minutes), occasional dropped runs, and public-repo 60-day-inactivity auto-disable (with warning e-mail + one-click re-enable). Verdict: unacceptable as the *primary* freshness guarantee, perfectly acceptable as the *backup lane* (D-02). healthchecks.io covers the silent-disable case.
- **N-05 — Poor man's event log (kept):** `observations` append-only per semantic change already gives an event history exactly where history matters. Real event sourcing (rebuilding state from events) rejected — cost, zero return. The optional suggestion tied to this section was **outbox-lite**, and it IS adopted (D-05).
- **N-06 — Neon sizing guard:** 0.5 GB / 100 CU-h free. Safe because of change-only writes (D-12) and the raw-HTML placement rule (local disk / Actions artifacts, never Neon).
- **N-07 — Model economics:** LJ measured DeepSeek V4 Flash ~5.4× cheaper than Gemini Flash-Lite on output tokens; owner's default order stands but is DB-config; the eval set (D-16) decides the final order with data.
- **N-08 — Model naming corrections:** "Gemini 3.7 Flash-Lite" does not exist (Lite is a separate family); "GPT-5.6 Luna Max" is `gpt-5.6-luna` + `reasoning effort: max`. All model IDs re-verified at integration time; never hardcoded.
- **N-09 — Anti-bot hygiene (unanimous):** robots.txt respected, per-domain rate limit + jitter (no fixed-time cron signatures), realistic UA, ETag/If-Modified-Since, block images/fonts in browser tier, consecutive-failure counter → auto-pause adapter + operator alert (adapter `health_score`).
- **N-10 — ADR template addition (LJ/P1):** every ADR carries a **"reopen signal"** field ("we revisit this decision when X happens"). This ledger's last column mirrors it.

---

## 3. Open micro-decisions

| ID | Question | Recommendation |
|---|---|---|
| O-01 | i18n scope: UI only, or also bot messages + notifications in 3 languages? | UI-only in MVP; `locale` column ready; bot/notifications TR (fallback EN) until MVP exit |
| O-02 | Is cross-site comparison of the same product (one item, N sites) an MVP requirement? | Schema already supports it (D-13); comparison UI = post-MVP epic; no auto-matching ever |
| O-03 | Monthly AI hard-budget number for the D-16 stop? | $5/month to start |
| O-04 | Privacy stance on sending scraped page content to Gemini/DeepSeek? | Acceptable: public product pages only; no personal data in prompts; log purpose per call |
| O-05 | Weekly hours available + target date for the first real notification (roadmap calibration)? | Owner input needed |
| O-06 | Which OEM brand is the MVP pilot (e.g., Toyota TR price list)? | Owner picks 1; PDF-capable adapter built against it |

---

## 4. Assumptions on record

- Single user, single workspace throughout MVP; Telegram is the primary notification channel.
- Primary machine is expected to be on most of the day; gaps are covered by the backup lane; both lanes silent → healthchecks alert.
- Hourly checks may be delayed 15–30 min on the backup lane — acceptable for price tracking.
- No paid infrastructure in MVP ($0 target); first acceptable spend: ~$10/mo proxy only at the sahibinden reopen (D-07), ~$5/mo AI budget (O-03).
- Task ID prefix: `PT-###`.
Karar defteri oluşturuldu: docs/decision-ledger.md — 27 kilitli karar (D-01..D-27), 10 kayıtlı not/itiraz (N-01..N-10), 6 açık mikro-karar (O-01..O-06). b-1/b-2/b-3/b-4/b-5/b-6 kararlarının hepsi işlendi. Şimdi a-1 ve a-2 raporu:

a-1. GPT ile hemfikir olduğumuz alanlardan aldığım iyileştirmeler
Bunlar çatışma değil — aynı yöndeki görüşün daha iyi ifade edilmiş/rafine hali. Hepsini deftere işledim:

run-due-checks idempotent worker komutu (→ D-26): scrape→compare→persist→notify tek komut; ev scheduler'ı da, Actions backup lane'i de, manuel /check de aynı komutu çağırır. Benim iki-hat tasarımımın daha temiz omurgası; ayrıca "JobQueue portunu 1. günden tanımla" (sketch/jellyfish) fikrini gereksizleştiriyor — dikiş yeri artık bu komut, kuyruk arayüzü kuyruk gelince gelir.
Foundation'dan ÖNCE 1-2 günlük risk spike'ı (→ D-27): iki pilot kaynaktan gerçekten güvenilir veri çekilebiliyor mu, hiçbir altyapı kurmadan doğrula. Walking-skeleton'dan bile önce, en ucuz de-risk adımı.
Fixture testi semantiği düzeltmesi (→ N-03, D-17): kayıtlı HTML fixture'ı canlı sitenin değiştiğini tespit edemez (fixture donmuş) — sadece kodun bilinen HTML'e karşı gerilemesini yakalar. Site değişimini gece çalışan live-smoke yakalar. Sketch/jellyfish (ve ben onları aktarırken) fixture'a fazla anlam yüklemiştik; düzeltildi.
Eval seti (20-50 gerçek örnek) model sırasını kilitlemeden önce (→ D-16). Jellyfish'in "DeepSeek aslında en ucuz, ölç" notuyla birleşiyor.
İki provider yeter, üçüncüsü config (→ D-16): Gemini + DeepSeek adapter'ı yaz, üçüncü sağlayıcı zaten sadece konfigürasyon.
workspace_id sadece aggregate root'larda (→ D-10): sketch'in "her tabloda" kuralı yerine GPT'nin rafinesi — child tablolar parent üzerinden erişir, körlemesine kolon tekrarı gürültü.
Ham HTML asla Neon'a yazılmaz (→ D-12): Neon free 0.5GB; hata anındaki HTML primary'de lokal diske (7 gün rotasyon), backup lane'de Actions artifact'ine.
Yalın monorepo yerleşimi (→ D-04): boş landing/pm/mobile app'leri scaffold edilmez; bot api içinde modül, worker ince entrypoint; native Tauri fazı başlayınca açılır.
AI extraction karantinası (→ D-06): AI'dan çıkan fiyat asla doğrudan alert tetiklemez — confidence ile şüpheli işaretlenir, insan onayı ister; asıl işi parser-onarım taslağı üretmek. (Jellyfish'in sanity guard'larıyla birleşti.)
"Actions scheduler olamaz" ifadesinin yumuşatılması (→ N-04): sketch fazla kesindi, GPT haklı — "SLA scheduler'ı değil ama best-effort backup lane olarak meşru". Zaten nihai tasarımımız tam bu.
a-2. Çatışma raporu: pozisyonumu koruyorum / geri adım / senin kararın
Geri adım attıklarım (GPT haklı)
ADR sayısı — GPT: "başta 5-7 çekirdek ADR; 18 ADR yazıp koda başlamamak". Benim planım ~15+ ADR'ye gidiyordu. Katılıyorum: 7 çekirdek ADR (DB, runtime+hatlar, ORM, monorepo, bot modu, scraping stratejisi, para/zaman) kod öncesi yazılır; geri kalan her karar zaten karar defterinde — dokunulduğunda ADR'ye terfi eder (→ D-22).
AGENTS.md canonical, CLAUDE.md sadece araç-farkları — GPT haklı; sen fiilen çok araçlı çalışıyorsun (Kiro + Claude + GPT). CLAUDE.md kopya olmayacak (→ D-22).
Coverage kapısı — jellyfish'in "domain ≥%80" kapısını aktarmıştım; GPT'nin "erken global kapı metrik oyununa döner" itirazı doğru. Kapı sadece parser/domain paketlerinde, repo-geneli yüzde yok (→ D-17).
Blind review'ın girdisi — GPT/P1/P2: "salt diff gören reviewer beklenen davranışı bilemez; acceptance criteria göster, tasarım gerekçesini gizle." Doğru — senin 2-prompt kararınla çelişmiyor, "git changes review" promptunun şablonuna AC özeti eklenir (→ D-19).
Task-bazlı AI routing (hafif) — tek global zincir önermiştim; GPT'nin görev-bazlı yaklaşımı (normalize=ucuz; sıralama=ucuz, düşük güvende yükselt; parser-onarımı=güçlü+insan onayı) hafif haliyle daha iyi: default zincir durur, görev başına override eklenir (→ D-16).
Pozisyonumu koruduklarım
Git akışı — GPT: "korumalı main + kısa feat/* + PR + squash; dev'i sorgula." Bu benim ilk turdaki önerimdi zaten; ama sen main+dev/PR'sız modeli seçtin ve ben ikinci turda bunun senin faz-geçişi review akışınla tutarlı olduğunu gösterdim — o değerlendirmemin arkasındayım. Belirleyici: main..dev diff'i = review girdisi; solo + AI review'da GitHub PR arayüzünün katma değeri yok. GPT'nin akışı deftere alternatif olarak yazıldı, geri-dönüş sinyaliyle: "ortak geliştirici veya gerçek staging ortamı gelirse PR akışına geçilir" (→ D-20). Yeni karar gerekmez.
board.ts generator'ı — GPT: "board tek dosyayken manuel; drift oluşursa generator ekle." Koruyorum: ~60 satırlık script + lefthook'un maliyeti sıfıra yakın, doküman çürümesi ise AI-workflow'un 1 numaralı katili (sketch/jellyfish hemfikirdi) ve sen bunu zaten onaylamıştın. GPT alternatifi deftere not edildi (→ D-22).
Story point — GPT: "ilk 3 döngü SP yok, cycle time kaydet." Sen "tut" kararını vermiştin; koruyorum ama GPT'nin eklentisini alıyorum: 1. günden gerçek süre/cycle time da kaydedilir, ~3 döngü sonra kalibrasyon bakılır (→ D-18). Fiilen ikisinin birleşimi.
Base plan — GPT: "P1 en iyi base." Ben: "jellyfish iskelet + sketch süreç." Gerçekte yöntemde birleşiyoruz — GPT de "hiçbirini olduğu gibi alma, kararlar sonrası temiz sentez" diyor (senin onayladığın yol). Fark sadece üslup kaynağı; çözüm: editoryal yoğunluk P1'den (200-350 satır, ölçülü), teknik içerik jellyfish+sketch'ten, hepsi karar defteri üzerinden (→ D-27). Kavga yok, sentez netleşti.
Veri modeli / canonical product uyarısı — GPT: "URL'yi canonical product sayma, siteler-arası eşleştirme zor problem." Gerçek ihtilaf yok: zaten hiçbir planda otomatik eşleştirme yoktu (insan onayı kuralı). İki seviyeli şemayı (tracking_target → source_entries) koruyorum — şimdi bedava, ileride karşılaştırma ekranını migration'sız mümkün kılar; otomatik eşleştirme asla (→ D-13). Küçük bir ucu sana kalıyor: O-02.
Senin verdiğin, GPT'nin itiraz ettiği, benim uyguladığım kararlar (yeniden açmıyorum, sadece kayda geçtim)
MVP genişliği (OEM MVP'de + 3 kategori + PWA): GPT "en fazla 2 pilot adapter" der; sen OEM'i MVP'ye aldın (b-6-c). Uygulanır; risk sıralamayla yönetilir: walking skeleton (Vatan) → arabam → OEM; ilk gerçek bildirim uçtan uca çalışmadan hiçbir genişletme başlamaz (→ D-07). Kategori/liste takibi Faz 2'de kalıyor (senin 18. karardaki onayınla tutarlı — OEM'i aldın, kategori takibini almadın; doğru mu anladım, yanlışsa söyle).
i18n MVP'de (EN/TR/AR): 8 planın tamamı + GPT + ben YAGNI demiştik; owner olarak override ettin — uygulanır. Dürüst maliyet notu: Arapça = RTL, bu gerçek bir UI vergisi (~+%15-25 UI eforu). Mitigasyon: 1. günden logical properties (ms-/me-/ps-/pe-), Radix DirectionProvider, UI story'lerinin DoD'una "RTL'de kontrol edildi" maddesi; framework react-i18next (AI eğitim verisinde en yaygın) (→ D-23). Bir ucu açık: O-01.
b maddeleri — teyit
b-1/e: İkincil hat tasarımı (primary/secondary/staleness-check/healthchecks.io) aynen D-02'ye kilitlendi.
b-2: CF Workers analizi tam gerekçesiyle N-01'e; tüm GPT itirazları ve çözümleri N-02'ye; her birinin geri-dönüş sinyali ilgili kararın satırında.
b-3: "Neyi neden aldık/almadık" defterin ana formatı oldu (her satırda gerekçe + reddedilenler + reopen trigger). Sorduğun optional öneri: ES bölümünün yanındaki öneri outbox-lite idi (bildirim önce DB'ye pending yazılır, dispatcher gönderip sent yapar — crash'te bildirim kaybolmaz) — evet, alındı (→ D-05); "fakirin event log'u" tespiti de tasarım notu olarak N-05'te.
b-4: Tailscale (→ D-09) + public repo (→ D-24, macOS runner'lar bedava → iOS/macOS build matrisi $0).
b-5: main+dev modeli iki ayarıyla (dev push CI + commit disiplini) D-20'de.
b-6: Docs=EN (D-22), i18n MVP (D-23), OEM MVP'de (D-07), Tailscale (D-09).
Kalan açık mikro-kararlar (son tur — bunlar da gelince plan yazılır)
O-01 — i18n kapsamı: 3 dil sadece UI'da mı, bot mesajları + bildirimler de mi? (Önerim: MVP'de UI-only; locale kolonu hazır; bot/bildirim TR, fallback EN.)
O-02 — Siteler-arası karşılaştırma (aynı notebook, N site) MVP gereksinimi mi? (Önerim: şema destekliyor, karşılaştırma UI'ı post-MVP.)
O-03 — Aylık AI bütçe limiti (sert durdurma sayısı)? (Önerim: $5/ay.)
O-04 — Gizlilik: sayfa içeriğini Gemini/DeepSeek'e göndermek sorun mu? (Önerim: sorun değil — halka açık ürün sayfaları, prompt'ta kişisel veri yok.)
O-05 — Haftada kaç saat + ilk gerçek bildirim için hedef tarih? (Roadmap'i gerçekçi takvimlemek için — GPT'nin sorusu, bende de cevabı yok.)
O-06 — OEM pilot markası hangisi? (örn. Toyota TR fiyat listesi)
Bunları cevapladığında (veya "önerilerinle geç, O-05: X saat, O-06: Y" gibi): konsolide canonical plan (200-350 satır) + 7 çekirdek ADR + docs/prompts kit'i + AGENTS.md/CLAUDE.md/CONTEXT.md iskeletini tek seferde yazacağım — hepsi karar defterinden türeyecek.
Based on official Claude Code documentation, here is the definitive answer:

Definitive Answer
Claude Code does NOT automatically read an AGENTS.md file as project instructions or memory. Claude Code reads only CLAUDE.md files (and CLAUDE.local.md for personal preferences). The officially recommended way to use AGENTS.md content is to create a CLAUDE.md that imports it using the @AGENTS.md syntax: place @AGENTS.md anywhere in your CLAUDE.md, and Claude Code will expand and load the imported file at session start. Alternatively, you can create a symlink (ln -s AGENTS.md CLAUDE.md) on Linux/macOS, though Anthropic recommends the @ import method on Windows due to permission requirements. The @path/to/file import mechanism for CLAUDE.md is an official, currently active feature documented in the memory.md page. Imports are fully recursive (up to 4 hops deep) and resolved relative to the file containing the import. When both AGENTS.md and CLAUDE.md exist, only CLAUDE.md loads automatically; you must explicitly import AGENTS.md into CLAUDE.md for it to be read. The /init command (with CLAUDE_CODE_NEW_INIT=1 environment variable enabled) can read and incorporate AGENTS.md during setup, and it also recognizes other agent formats (.cursorrules, .clinerules, etc.), but this is a one-time setup step, not continuous auto-loading.

Version and Date Facts
@-import syntax: Introduced in Claude Code v0.2.107; currently stable and documented in all v2.1.x releases (confirmed in the current documentation at code.claude.com, latest v2.1.239, August 21, 2026).
AGENTS.md support: No dedicated "AGENTS.md auto-load" feature exists; support is via the general @-import mechanism and the /init command's ability to read and incorporate AGENTS.md during initial setup. The memory.md documentation does not specify a version when AGENTS.md import support was added, but it is explicitly documented and current as of August 2026.
No special settings/env flags: AGENTS.md handling requires no special flags. The @AGENTS.md syntax works in any CLAUDE.md without configuration. For the interactive setup flow that can read AGENTS.md, set CLAUDE_CODE_NEW_INIT=1 when running /init.
Source URLs Verified
Claude Code Memory Documentation — official docs covering CLAUDE.md, AGENTS.md, @-imports, and load behavior
Claude Code Docs Map — index confirming the memory.md page
GitHub CHANGELOG excerpt — v0.2.107 introduced @-import syntax
Ajan doğruladı: AGENTS.md otomatik okunmuyor — CLAUDE.md içine @AGENTS.md import'u şart. Buna göre CLAUDE.md'yi yazıyor, deftere işliyor ve kalan dosya setini üretiyorum.

Edit [repo]\docs\decision-ledger.md
Modified
Write [repo]\AGENTS.md
65 lines
# AGENTS.md — Repo Contract for AI Agents

> Canonical instructions for **every** AI tool working in this repo (Kiro IDE, Claude Code,
> Codex/GPT, Cursor, …). Tool-specific deltas live in that tool's own file (e.g. `CLAUDE.md`,
> which imports this one) and must never duplicate it.

## What this project is

**AI Price Tracker** — tracks prices/stock of products on Turkish sites (notebooks, used cars on
arabam.com, new-car OEM price lists: Kia Sportage / Toyota RAV4), detects semantic changes, and
notifies via Telegram; dashboard as web SPA + PWA, later Tauri desktop/Android/iOS. Solo project,
$0 infrastructure, built through AI seasons with a heavy emphasis on process artifacts.

## Read first, in this order

1. [docs/CONTEXT.md](docs/CONTEXT.md) — invariants (hard 200-line cap). **Non-negotiable.**
2. [docs/canonical-plan.md](docs/canonical-plan.md) — architecture & process. All decided.
3. [docs/decision-ledger.md](docs/decision-ledger.md) — every decision with rationale, rejected
   alternatives, and reopen triggers (D-xx / N-xx / O-xx).
4. The active spec under `.kiro/specs/PT-###-slug/` when implementing or reviewing.

## Ground rules

- **Knowledge flows through files, never chat memory.** If it isn't written, it doesn't exist.
- **Decisions are locked in the ledger.** Do not relitigate or silently deviate; to challenge a
  decision, name the D-xx and its reopen signal.
- **AI proposes, human approves.** Never auto-track a product; never auto-apply parser repairs;
  AI-extracted prices are quarantined and never fire alerts unconfirmed.
- **No CAPTCHA/login/anti-bot bypass code — ever.** Such sources are marked `unsupported`.
- **Secrets never in code or docs.** GitHub Secrets / gitignored `.env` only.

## Conventions

- Code EN · docs EN · user-facing strings only via i18n catalogs (**EN/TR/AR** — UI, bot replies,
  and notifications; UI must be RTL-safe).
- Money = bigint minor units + ISO-4217 currency, never float. Time = UTC `timestamptz`.
- Commits: Conventional Commits + task id — `feat(scraper): vatan adapter (PT-014)`.
  Every commit leaves the repo working.
- Branches: work on `dev`; `main` only via phase-gate merges. **No PRs, no feature branches.**
- Nx module boundaries are law; `packages/domain` depends on nothing.
- Specs only for work above a few hours or spanning layers; small work = board entry + commits.

## Season protocol (planner / implementer / reviewer)

- **Planner** produces the spec triplet (`requirements.md` EARS / `design.md` / `tasks.md`) in
  `.kiro/specs/PT-###-slug/` — prompt: [docs/prompts/planner-spec.md](docs/prompts/planner-spec.md).
- **Implementer** receives ONE task + `design.md` + `CONTEXT.md`; deviations go to `handoff.md`,
  never silently into code — prompt: [docs/prompts/implementer.md](docs/prompts/implementer.md).
- **Reviews** run at phase gates in fresh sessions: informed review
  ([docs/prompts/informed-review.md](docs/prompts/informed-review.md)) + independent
  git-changes review ([docs/prompts/git-changes-review.md](docs/prompts/git-changes-review.md)).
  An implementer never reviews its own work in the same session.

## Commands

Phase 0 will fill this section (pnpm/nx/docker/test commands). Until then there is nothing to run.

## Repo map

`apps/` — api (NestJS + grammY bot module), worker (scheduler → `run-due-checks`), web
(React SPA/PWA); `native` appears at the Tauri phase.
`packages/` — contracts, domain, db, scraping, ai, notifications, testing.
`docs/` — canonical plan, ledger, ADRs (`decisions/`), prompts, roadmap, runbooks, work/board.
`.kiro/` — steering + specs (generated via Kiro IDE).
Write [repo]\CLAUDE.md
13 lines
# CLAUDE.md

@AGENTS.md

Claude-specific deltas only — the imported contract above is canonical:

- Reviews at phase gates run in **fresh sessions**. For the independent review, use
  `docs/prompts/git-changes-review.md` and do **not** open `.kiro/specs/*/design.md`,
  `handoff.md`, or any chat history — only the inputs the prompt allows.
- Project-specific skills are a deferred backlog item (ledger D-25); until they exist, follow
  AGENTS.md + `docs/prompts/` directly.
- Scratch/temp output goes to the session scratchpad, never into the repo.
Write [repo]\docs\CONTEXT.md
65 lines
# CONTEXT.md — Invariants

> **HARD CAP: 200 lines.** Every AI season reads this file first. If a change conflicts with
> anything here, stop and flag it (name the ledger D-xx). If this file needs to grow past the cap,
> split details into `docs/architecture/` and trim.

## Data

- Money = **bigint minor units (kuruş)** + ISO-4217 `currency` column. Float never.
  API serializes money as string.
- Time = **UTC `timestamptz`** everywhere; localize only at presentation.
- `workspace_id` on aggregate roots only; single seed workspace in MVP.
- `observations` are written **only on semantic change** (`snapshot_hash` differs); otherwise
  update `last_checked_at`. Every attempt is logged to `scrape_runs` (with `lane`).
- Notifications are outbox-lite: insert `pending` → dispatcher sends → `sent`.
  `dedupe_key` + cooldown: the same change never notifies twice.
- Raw HTML (failure debugging) → local disk (7-day rotation) or Actions artifact — **never Neon**.
- Soft-archive with `archived_at`; never hard-delete tracking history.

## Scraping

- Ladder order is mandatory: official API → fetch → JSON-LD/embedded state → Cheerio adapter →
  PDF (unpdf) → Playwright **last**.
- **Never bypass CAPTCHA/login/anti-bot.** Mark such sources `unsupported`.
- Per-domain rate limit + jitter; robots.txt respected; realistic UA; ETag/If-Modified-Since.
- `parse()` is **pure**: recorded input → Observation. Network-free, fixture-tested.
- Sanity guards: price ≤ 0 or a >70% swing ⇒ mark suspicious, hold the alert, ask the human.
- AI-extracted values are quarantined (`extraction_method='ai'`): never alert unconfirmed.
- Fixture tests catch **code regressions only** — live-site drift is caught by the nightly
  live-smoke job, not by fixtures.

## AI

- **AI proposes, human approves** — no auto-tracking, no auto-applied parser repairs.
- All AI outputs are Zod-validated; retry with schema feedback, then fall back to the next
  provider; circuit breaker on consecutive failures.
- Model IDs live in DB config, never in code. Every call logged to `ai_calls`
  (purpose, tokens, latency, cost).
- No personal data in prompts; public product pages only.

## API & contracts

- Single **`/api/v1`**. Zod schemas in `packages/contracts` are the single source of truth;
  OpenAPI is derived from them and snapshot-tested in CI.
- All HTTP goes through the shared fetch wrapper (Zod-validated responses; the transport
  auto-swaps to `@tauri-apps/plugin-http` inside Tauri builds).

## i18n

- **EN/TR/AR on ALL user-facing text**: UI, bot replies, notifications — via react-i18next
  catalogs keyed by the user's `locale`.
- UI must be **RTL-safe**: logical properties (`ms-/me-/ps-/pe-`, start/end), dir-aware
  components. Every UI story's DoD includes an RTL check.
- Never hardcode a user-facing string.

## Process & git

- Work on `dev`; merge to `main` only at phase gates. Conventional Commits + `PT-###`.
  Every commit leaves the repo working.
- Specs only for work above a few hours or spanning layers. Deviations from a spec go to
  `handoff.md` — never silently into code.
- **No-Fiction:** artifacts are generated from real data; skipped ceremonies/milestones get a
  short doc stating why they were skipped.
- Secrets are never committed.
Write [repo]\docs\roadmap.md
88 lines
# Roadmap

> Phase-based and **date-free** (owner decision O-05). Each phase ends at an **evidence gate**:
> every criterion is answered with proof — command output, screenshot path, DB query result, or
> commit SHA. No evidence, no gate. **Iron rule (D-07):** nothing broadens until Phase 1's first
> real notification works end to end.

## Phase −1 — Risk spike (1–2 days, throwaway code)

Prove the pilots are extractable **before** building any foundation:

- Vatan product page: price/stock via plain fetch + JSON-LD/embedded state?
- arabam.com listing page: price/status extractable? Which ladder rung?
- Kia TR (Sportage) + Toyota TR (RAV4) price lists: HTML or PDF? Is `unpdf` viable?

**Gate:** `docs/records/spike-results.md` — table: source → ladder rung → sample extracted
values. Unsupported findings recorded honestly.

## Phase 0 — Foundation

- Nx + pnpm workspace; `apps/{api,worker,web}` skeletons; `packages/*` with boundary rules.
- `docker-compose` (Postgres) + Zod-validated env config.
- Drizzle schema v1 + first migration (canonical plan §4 tables).
- Two-tier CI: `dev` push → lint/typecheck/unit/integration (Testcontainers);
  `main` merge → e2e/build.
- Docs live: AGENTS/CLAUDE/CONTEXT, ledger, ADR 0001–0007, prompts, `board.ts` + lefthook,
  light `pm-check` script.
- `.kiro/steering/` seeded from CONTEXT.md + canonical plan.
- healthchecks.io check created; bot token & keystore secrets in GitHub Secrets.

**Gate:** `pnpm nx run-many -t lint,typecheck,test` green locally and in CI ·
`/api/v1/health` → 200 · empty dashboard renders in LTR **and** RTL · board generated.

## Phase 1 — Walking skeleton

One thin slice, no beauty: Telegram `/add <vatan-url>` → adapter → preview → approve →
target + entry + first observation → scheduler runs `run-due-checks` → price change →
Telegram alert. Backup lane workflow live (no-op path verified); healthchecks pinging.

**Gate:** a real price row in Neon · a fixture-forced change produces exactly **one** alert ·
a repeat run produces **none** (dedupe) · backup-lane no-op visible in `scrape_runs` ·
healthchecks shows pings · CI green.

## Phase 2 — MVP build-out (strict order)

1. arabam.com adapter (used-car listing URLs).
2. OEM adapters: Kia TR (Sportage), Toyota TR (RAV4) — PDF-capable, daily cadence.
3. Alert rules (target price, % drop, back-in-stock) + cooldown; rule editor.
4. Dashboard: `dashboard-01` shell · items table · item detail with price-history chart ·
   **cross-site comparison view** (one target ↔ N source entries; linking human-approved) ·
   add flows · settings (AI routes).
5. AI discovery (free text → candidates → approval) + eval set (20–50 real samples) + `/model`.
6. **i18n:** EN/TR/AR catalogs across UI + bot + notifications; full RTL pass.
7. **PWA:** manifest + service worker (web profile only) + Tailscale Serve HTTPS;
   real-phone Home-Screen install.

**Gate (MVP exit):** 10+ targets across 4 sources tracked for 2 weeks · alert accuracy ≥ 90% ·
nightly live smoke green (or its failure alert demonstrably fired) · locale switch works in all
3 languages **including bot replies and a notification** · PWA installed on a real phone via
Tailscale · phase review done with both prompts + reconciliation record.

## Phase 3 — Native

Tauri desktop (Windows locally; macOS/Linux via CI) → Android **signed APK** (personal keystore;
`adb install` on a real phone; no store) → iOS **build + simulator smoke only** (CI macOS —
free on the public repo). `tauri-driver` smoke set.

**Gate:** desktop artifacts for 3 OSes in CI · APK completes core flows on a real phone ·
iOS simulator smoke green.

## Phase 4 — Post-MVP / hardening

Category/list-URL tracking epic (new/removed listings, pagination caps, removal confirmed after
2 consecutive misses) · more adapters · Crawlee **if** list crawling demands it ·
sahibinden.com spike (**only if** ~$10/mo proxy budget approved AND MVP stable for 1 month) ·
bot → Cloudflare Workers webhook **if** the D-03 trigger fires (uptime < 95%/mo or external
command need) · backup/restore runbook drill · optional DuckDB analytics experiment.

## Phase 5 — SaaS-readiness

Real auth (pairing/sessions per the P2 design, or OAuth) · quotas/limits · Astro landing
(separate app) · billing research · KVKK/GDPR texts · ToS/legal review + affiliate-API path
before any scraping-as-a-service.

## Phase 6 — Enterprise (on real demand only)

SSO/SCIM · RBAC · audit log · SLA/SLO · DR/backup policy · compliance.
Şimdi 7 çekirdek ADR:

Write [repo]\docs\decisions\ADR-0001-postgresql-from-day-one.md
32 lines
# ADR-0001 — PostgreSQL from day 1

- **Status:** Accepted · **Date:** 2026-08-22 · **Ledger:** D-01

## Context
Greenfield project; scraper, API, and bot write to the DB concurrently; a SaaS path exists;
Neon's free Postgres tier is available; Drizzle is the ORM (ADR-0003).

## Decision
PostgreSQL from the first migration. Neon (runtime) + Docker Postgres (local dev & integration
tests). No SQLite/Turso stage, no dual-dialect configuration.

## Rationale
1. Drizzle schemas are dialect-specific (`sqliteTable` ≠ `pgTable`) — "migrate later" means
   rewriting schemas, migrations, and type tests. It is not a driver swap.
2. SQLite's single-writer lock (`SQLITE_BUSY`) is exactly this workload's weak spot
   (3 concurrent writers).
3. Type loss in SQLite: `timestamptz`, `jsonb`, partial indexes, window-function-friendly
   time series.
4. A cloud-reachable DB is what makes the dual-lane runtime (ADR-0002) possible at all.

## Rejected alternatives
- **SQLite / Turso start** (recursive-lamport, plan-1): migration cost understated.
- **Dual dev-SQLite / prod-PG** (replicated-crayon): two schema sources to keep in sync forever.

## Consequences
(+) One dialect forever; both lanes reach the same DB. (−) Docker required locally; Neon free
caps (0.5 GB / 100 CU-h) — mitigated by change-only writes and the no-raw-HTML-in-DB rule (D-12).

## Reopen signal
Neon free limits exceeded → self-hosted Postgres on the home server or a paid tier.
Write [repo]\docs\decisions\ADR-0002-dual-lane-runtime.md
37 lines
# ADR-0002 — Dual-lane runtime: home server primary + GitHub Actions backup

- **Status:** Accepted · **Date:** 2026-08-22 · **Ledger:** D-02, D-26, N-01, N-04

## Context
$0 infrastructure target. Scraping success depends heavily on IP reputation (residential IPs are
blocked far less than datacenter IPs). GitHub Actions scheduled workflows are best-effort:
queue jitter of minutes, occasional dropped runs, and public-repo scheduled workflows are
auto-disabled after 60 days of repo inactivity (warning e-mail + one-click re-enable).
Free PaaS options are effectively gone (Railway $1/mo credit, Render cold starts, Fly no free tier).

## Decision
1. **Primary:** home server (Docker) — in-process scheduler; every successful run logged to
   `scrape_runs` with `lane='primary'`.
2. **Backup lane:** the same worker CLI on a 60-min Actions cron with `SCRAPE_LANE=backup`.
   First step queries the DB: if the last primary run is < 3 h old → no-op exit (~10 s).
   If stale → scrape, write to Neon, notify via Telegram `sendMessage` (bot token in GH Secrets).
   Lanes self-coordinate through DB timestamps — no double scraping or double notifications.
3. **Dead-man's switch:** healthchecks.io — primary pings on every successful cycle; missing
   pings for X h → alert. Covers "silent vs. quiet" ambiguity AND the Actions 60-day disable.
4. **Single scheduling surface:** one idempotent `run-due-checks` command
   (scrape → compare → persist → notify), called by the home scheduler, the backup lane, and
   manual `/check`. No queue framework until a queue is real.

## Rejected alternatives
Actions as primary scheduler (lamport/grove/plan-2) · Railway/Render/Fly · a CF Workers
"standby bot" (impossible: Telegram polling XOR webhook, N-01) · BullMQ/Redis now.

## Consequences
(+) $0; residential IP for the hard sites; self-healing freshness. (−) Bot commands are offline
while the primary is down (accepted risk); the backup lane runs from datacenter IPs and can only
cover the easy sources.

## Reopen signal
Primary uptime < ~95%/month, or external command access becomes a need → **migrate** (not
back up) the bot to a Cloudflare Workers webhook (ADR-0005's reopen path).
Write [repo]\docs\decisions\ADR-0003-drizzle-orm.md
31 lines
# ADR-0003 — ORM: Drizzle

- **Status:** Accepted · **Date:** 2026-08-22 · **Ledger:** D-01 (context), unanimous across all 8 source plans

## Context
TypeScript monorepo; Zod-first contracts; PostgreSQL; price history is a time series
(window functions, partial indexes); AI seasons benefit from a small, readable data layer.

## Decision
Drizzle ORM, with migrations from the first commit (`drizzle-kit`; never `db push` to prod).
NestJS integration via our own ~20-line `DrizzleModule` provider — no third-party wrapper
package (abandonment risk eliminated).

## Rationale
SQL-first (generated SQL is readable; window functions natural); ~zero runtime overhead and no
codegen engine; `drizzle-zod` completes the single type chain **DB schema → Zod → OpenAPI → FE
types** — change a column and TypeScript breaks every stale consumer; migrations are plain SQL.

## Rejected alternatives
- **Prisma:** its own schema DSL breaks the single-source Zod chain; heavy engine/cold start.
- **TypeORM:** maintenance history, weak type safety, unreliable migration generation.
- **MikroORM:** genuine enterprise patterns (UoW/Identity Map) but conceptual weight a solo
  project does not need.

## Consequences
(+) One type chain end to end. (−) Relational query API is younger; no first-party Nest module
(solved by our own provider).

## Reopen signal
If heavy DDD aggregates become real → re-evaluate MikroORM.
Write [repo]\docs\decisions\ADR-0004-nx-pnpm-monorepo.md
29 lines
# ADR-0004 — Monorepo: Nx + pnpm

- **Status:** Accepted · **Date:** 2026-08-22 · **Ledger:** D-04

## Context
One repo hosts api/worker/web (+ native later) and 7 shared packages. Owner's explicit learning
goal: enterprise patterns. Architecture rot is the main long-term risk of a solo AI-driven repo.

## Decision
Nx with pnpm. `@nx/enforce-module-boundaries` lint rules from day 1:
`apps/*` → `packages/*` allowed; `packages/domain` depends on **nothing**; `packages/db` only on
`contracts`. Lean layout — thin entrypoints over shared services; no empty scaffold apps
(`landing`, `pm`, `mobile` are created only when their phase starts).

## Rationale
7 of 8 source plans chose Nx. Boundary rules are the only *automatic* mechanism protecting the
architecture — documentation alone rots. `nx affected` keeps CI fast and within free minutes.

## Rejected alternatives
- **Turborepo** (federated-grove): lighter, but no boundary enforcement/generators and weaker
  enterprise signal.
- **npm / yarn:** pnpm chosen (owner decision; also every source plan's assumption).

## Consequences
(+) Mechanical architecture protection; generators available later. (−) Nx config learning curve.

## Reopen signal
None foreseen.
Write [repo]\docs\decisions\ADR-0005-telegram-long-polling.md
33 lines
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
Write [repo]\docs\decisions\ADR-0006-layered-scraping.md
39 lines
# ADR-0006 — Scraping: layered ladder, Playwright last

- **Status:** Accepted · **Date:** 2026-08-22 · **Ledger:** D-06, D-07, N-03, N-09

## Context
Scraping is the project's riskiest subsystem: sites change HTML silently, anti-bot systems block
datacenter IPs aggressively, and $0 infrastructure rules out proxy stacks. Pilots: Vatan
(notebook), arabam.com (used car), Kia TR + Toyota TR price lists (PDF-capable).

## Decision
Extraction ladder, cheapest first — **0)** official API/feed → **1)** native fetch →
**2)** JSON-LD / meta / embedded state → **3)** site adapter + Cheerio → **4)** PDF (`unpdf`) →
**5)** Playwright only when forced.

- `SourceAdapter` port with a **pure** `parse()` (recorded input → Observation; fixture-tested).
- Hygiene: robots.txt, per-domain rate limit + jitter, realistic UA, ETag/If-Modified-Since,
  consecutive-failure counter → auto-pause + operator alert, adapter `health_score`.
- Sanity guards: price ≤ 0 or > 70% swing ⇒ suspicious — hold the alert, ask the human.
- **AI extraction quarantined:** `extraction_method='ai'` + confidence; never alerts
  unconfirmed; primarily drafts parser repairs (human-approved).
- **Test pairing (N-03):** fixture tests catch code regressions only — they cannot see live-site
  changes; the nightly live-smoke job (1 real URL/adapter, non-CI-blocking, failure → Telegram)
  catches drift.
- **Never bypass** CAPTCHA/login/anti-bot; such sources are marked `unsupported`
  (sahibinden.com deferred to Phase 4 on this basis).
- Crawlee deferred until category/list crawling is real.

## Rejected alternatives
Playwright-first (linear-lark) — 10–50× more expensive per check and doesn't beat anti-bot
anyway · Crawlee day 1 (tender-sketch, replicated-crayon) · stealth/proxy stack now
(replicated-crayon).

## Consequences
(+) Most checks are cheap and fast; honest "unsupported" list; home residential IP does the
heavy lifting. (−) Some sources stay out of reach until Phase 4 budget decisions.

## Reopen signal
List crawling ships → adopt Crawlee. sahibinden: ~$10/mo proxy budget AND 1 month stable MVP.
Write [repo]\docs\decisions\ADR-0007-money-and-time.md
32 lines
# ADR-0007 — Money and time representation

- **Status:** Accepted · **Date:** 2026-08-22 · **Ledger:** D-11

## Context
Every core feature compares prices (diffs, thresholds, percentages). Drizzle returns Postgres
`NUMERIC` as strings, a classic silent-bug source. Timestamps flow between scraper, DB, API, and
three UI locales.

## Decision
- Money = **`bigint` minor units (kuruş)** + ISO-4217 `currency` column. Never float, never
  NUMERIC. The API serializes money as a string; a `Money` value object handles arithmetic and
  formatting in `packages/domain`.
- Time = **UTC `timestamptz`** everywhere; localization only at the presentation layer.

## Rationale
Integer arithmetic makes diff/threshold math deterministic and eliminates the float error class
entirely; it also sidesteps Drizzle's numeric-string trap. UTC-everywhere avoids timezone drift
across lanes and clients.

## Rejected alternatives
- `NUMERIC(12,2)` + decimal strings (linear-lark, plan-2): valid, but string math in app code is
  less deterministic and error-prone.
- Float: never an option.

## Consequences
(+) Trivial, exact comparisons. (−) Conversion happens at the edges (parser input, UI display) —
owned by the `Money` value object.

## Reopen signal
Multi-currency aggregation needs (SaaS phase) → add FX tables; representation stays minor-unit.
Son parti: prompt kiti (5 dosya) + board tohumu:

Write [repo]\docs\prompts\planner-spec.md
35 lines
# Prompt — Planner / Spec Season (Kiro IDE or any planner session)

> Use when a story is above the spec threshold (more than a few hours, or spans layers — D-21).
> Fill the `<...>` placeholders, paste into a **fresh** planner session.

---

You are the **planner** for story `<PT-### — title>` in the AI Price Tracker repo.

**Read first (in order):** `docs/CONTEXT.md` (invariants — non-negotiable),
`docs/canonical-plan.md` (relevant sections), `docs/decision-ledger.md` entries `<D-xx list>`,
and the story description below.

**Story:** <one paragraph: user value, motivation, constraints>

**Produce three files under `.kiro/specs/PT-###-<slug>/`:**

1. `requirements.md` — EARS acceptance criteria:
   - `WHEN <trigger>, THE SYSTEM SHALL <behavior within bound>`
   - `IF <error condition>, THE SYSTEM SHALL <handling>`
   - Cover the unhappy paths (timeouts, empty/garbage input, dedupe, i18n/RTL where UI-facing).
2. `design.md` — interfaces (TypeScript signatures), data delta (tables/columns touched),
   error handling, and a **Test plan** section (which layers, which fixtures, which properties).
   No code bodies.
3. `tasks.md` — atomic tasks, each with:
   - single-file (or narrow) focus, referencing `design.md` sections,
   - a **verify-by** command (`pnpm nx test <project> -- <filter>`),
   - checkbox format.

**Rules:**
- Do not relitigate locked decisions; if a decision blocks the design, name the D-xx and stop.
- Prefer the smallest design that satisfies the ACs (YAGNI list in canonical plan §11 is binding).
- Every user-facing string in the design goes through i18n catalogs (EN/TR/AR), never hardcoded.
- Output only the three files, no commentary.
Write [repo]\docs\prompts\implementer.md
24 lines
# Prompt — Implementer Season

> One task at a time. Narrow context is the whole point — do not paste the full spec set.

---

You are the **implementer** for task `<T# from tasks.md>` of story `<PT-###>`.

**Your only inputs:** the task text below, `.kiro/specs/PT-###-<slug>/design.md`,
the acceptance criteria in `requirements.md`, `docs/CONTEXT.md`, `AGENTS.md`, and the repo code.

**Task:** <paste the single task from tasks.md, including its verify-by command>

**Rules:**
1. Implement exactly this task — no scope beyond it, no refactors "while you're here".
2. Do not deviate from `design.md`. If you must, or you hit a decision gap: record it in
   `.kiro/specs/PT-###-<slug>/handoff.md` (template: `docs/prompts/handoff-template.md`) and,
   for decision gaps, **stop and ask** instead of assuming.
3. Write the tests the task's verify-by implies; run the verify-by command; paste its real output
   into `handoff.md`. No fabricated results — No-Fiction applies to you too.
4. Respect every invariant in `CONTEXT.md` (money, time, i18n catalogs, ladder, quarantine…).
5. When green: tick the checkbox in `tasks.md`, update `handoff.md`, and commit as
   `<type>(<scope>): <what> (PT-###)`. The repo must work after your commit.
Write [repo]\docs\prompts\informed-review.md
27 lines
# Prompt — Informed / Conformance Review (phase gate, fresh session)

> Run in a FRESH session at a phase transition (D-19). This reviewer knows the intent.
> Never the same session that implemented the work.

---

You are the **informed reviewer** for phase `<N>` of the AI Price Tracker repo.

**Inputs:** for each story in this phase — `.kiro/specs/PT-###-*/requirements.md`, `design.md`,
`handoff.md` — plus the phase diff (`git diff main..dev`, or the commit range `<range>`)
and the CI/test results.

**Questions to answer, per story:**
1. Does the code implement what the spec designed? Which acceptance criteria are unmet?
2. Are the deviations recorded in `handoff.md` legitimate? Any **unrecorded** deviations?
3. Which error paths from `design.md` are missing or untested?
4. Do the CONTEXT.md invariants hold in the touched code (money, time, dedupe, i18n, quarantine)?

**Output:** append an `## Informed review` section to each story's
`.kiro/specs/PT-###-*/review.md`:
- Findings as `P0` (blocks the gate) / `P1` (must fix this phase) / `P2` (should fix) /
  `P3` (note), each with **evidence** (file:line, failing AC id, or command output).
- End with a verdict line: `GATE: pass | pass-with-P1s | fail`.

Do not fix anything; review only.
Write [repo]\docs\prompts\git-changes-review.md
46 lines
# Prompt — Independent "Git Changes" Review (phase gate, fresh session)

> Run in a FRESH session — ideally a different model family than the implementer (D-19).
> This reviewer does NOT know the design intent. Do not open `design.md`, `handoff.md`,
> the canonical plan, or any chat history. Allowed inputs are listed below — nothing else.

---

You are an **independent reviewer**. Assume this code is broken; your job is to find where.

**Your only inputs:**
1. Neutral goal, one line: <e.g. "price/stock tracker: scrape → detect change → notify via Telegram">
2. Acceptance criteria list (verbatim, no rationale): <paste the AC lines from requirements.md>
3. The diff: `git diff main..dev` (or commit range `<range>`)
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

## After both reviews: reconciliation (operator or a third short session)

Merge the informed + independent findings into `review.md` under `## Reconciliation`:
- **Agreements** (fix list, ordered), **Disagreements** (kept verbatim — these calibrate the
  owner's judgment; never average them away), **Verdicts:** `must-fix / should-fix / wont-fix
  (+reason)`. The phase gate closes only when every must-fix is done.
Write [repo]\docs\prompts\handoff-template.md
33 lines
# Template — handoff.md (implementer → next season)

> Lives at `.kiro/specs/PT-###-<slug>/handoff.md`. Updated by every implementer session.
> No-Fiction: real commands, real outputs, real gaps. An empty "deviations" section is a claim —
> make sure it is true.

```markdown
# Handoff — PT-### <title>

## Session <n> — <date>

### What I did
- <task ids ticked, one line each>

### Deviations from design.md (and WHY)
- <none | design said X, I did Y because Z — flagged for informed review>

### Decision gaps hit
- <none | question that needs the owner/planner, and what I did instead (stopped / minimal stub)>

### Known gaps / debt
- <not-covered edge case, skipped test, TODO left — with location>

### Commands run + real output (trimmed)
- `pnpm nx test <project> -- <filter>` → <pass/fail summary, pasted>

### Files touched
- <path list>

### Suggested next step
- <one line>
```
Write [repo]\docs\work\board.md
13 lines
# Board

> From Phase 0 on, this file is **generated** by `tools/board.ts` from story front-matter
> (lefthook pre-commit refreshes it) — do not edit by hand after that lands.
> WIP limit: **1** (In Progress).

| Backlog | Ready | In Progress (≤1) | In Review | Done |
|---|---|---|---|---|
| PT-001 Phase −1 risk spike (Vatan / arabam / Kia / Toyota extractability) | | | | |
| PT-002 Phase 0 foundation | | | | |

_Seeded 2026-08-22. First real population happens when Phase −1 starts._
Tamamlandı. Önce sorunun cevabı, sonra teslim edilenlerin özeti:

1. AGENTS.md sorusunun cevabı (web'den doğrulandı)
Hayır — Claude Code AGENTS.md'yi otomatik OKUMAZ. Resmi dokümantasyona (code.claude.com/docs/en/memory, güncel v2.1.x, Ağustos 2026) göre Claude Code yalnızca CLAUDE.md (ve CLAUDE.local.md) dosyalarını otomatik yükler. Resmi önerilen yöntem: CLAUDE.md içine @AGENTS.md satırı koymak — import mekanizması v0.2.107'den beri resmi ve güncel, özyinelemeli (4 seviyeye kadar) çalışıyor. Symlink alternatifi var ama Windows'ta izin gereksinimleri yüzünden önerilmiyor; /init komutu AGENTS.md'yi kurulum sırasında bir kez okuyabiliyor ama bu sürekli yükleme değil. Buna göre kurdum: AGENTS.md canonical sözleşme, CLAUDE.md ilk satırında @AGENTS.md + sadece Claude'a özel farklar. Mekanizma karar defterine D-22'ye kaynağıyla işlendi.

2. Kararların işlendi (2–7)
Deftere kilitlendi: i18n bot mesajları + bildirimler dahil EN/TR/AR (D-23) · siteler-arası karşılaştırma MVP'de (D-13, insan-onaylı eşleme, otomatik matching asla) · AI bütçe limiti yok — subscription'dan; ai_calls maliyet logu kalıyor, pay-per-use API key'e geçilirse limit geri gelir (D-16) · gizlilik onayı (O-04) · takvim planı yok — roadmap faz-bazlı, tarihsiz (O-05) · OEM pilotları: Kia Sportage + Toyota RAV4 (D-07). Açık karar kalmadı — O-01..O-06'nın tamamı çözüldü.

3. Söz verilen sentez teslim edildi — oluşturulan dosyalar
Dosya	İçerik
docs/canonical-plan.md	Tek yetkili plan (~290 satır): ürün, mimari diyagram, kilitli stack, veri modeli, scraping, AI, API/istemciler, test, süreç, YAGNI, doğrulama ritüeli — her madde D-xx referanslı
docs/roadmap.md	Tam yol haritası, Faz −1 (risk spike) → Faz 6 (Enterprise), her fazda kanıt-zorunlu çıkış kapısı, tarihsiz
docs/decisions/	7 çekirdek ADR (Postgres, çift-hat runtime, Drizzle, Nx+pnpm, long-polling, scraping merdiveni, para/zaman) — her birinde reddedilenler + geri dönüş sinyali
docs/prompts/	5 prompt: planner-spec (Kiro için), implementer, informed-review, git-changes-review (bağımsız + adversarial eklentisi + reconciliation), handoff şablonu
AGENTS.md	Canonical repo sözleşmesi (tüm AI araçları için)
CLAUDE.md	@AGENTS.md import + Claude'a özel 3 delta
docs/CONTEXT.md	Invariant'lar, 200 satır sert limitli (~75 satır kullanıldı) — her seansın ilk okuduğu dosya
docs/work/board.md	Board tohumu (PT-001 spike, PT-002 foundation)
docs/decision-ledger.md	Güncel: 27 karar + 10 not, tüm mikro-kararlar çözülmüş
Dikkatini çekmek istediğim iki içerik detayı: senin madde-9 sadeleştirmen prompt kitine birebir işlendi (faz geçişinde iki prompt; bağımsız review'a GPT'nin düzeltmesiyle AC listesi verilir ama tasarım gerekçesi verilmez), ve spec'lerin evi .kiro/specs/PT-###-slug/ olarak sabitlendi (Kiro üretir, handoff/review yanına yazılır).

4. Sıradaki adım
Planlama bitti; ilk iş Faz −1 risk spike'ı (PT-001): dört pilot kaynağın (Vatan, arabam, Kia TR, Toyota TR) gerçekten çıkarılabilir olduğunu 1-2 günlük atılabilir kodla kanıtlamak — hiçbir altyapı kurmadan. "Spike'ı başlat" dediğinde başlarım; istersen önce bu doküman setini gözden geçirip düzeltme isteyebilirsin. Repo'da henüz commit yok — bu iskeleti ilk commit olarak atmamı istersen onu da söyle (kendiliğinden commit atmıyorum).
