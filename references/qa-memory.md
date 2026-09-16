# QA memory — the `.qa/` folder

Without memory, every run starts as a stranger to the project: the same bugs get
re-found, the same intended behaviours get re-reported as bugs, and nothing
accumulates. A few small markdown files in `.qa/` at the repo root fix that. They
are cheap to maintain (Phase 6, two minutes) and they compound.

```text
.qa/
├── README.md              # the index: file map + suite table — Phase 0 starts here
├── critical-flows.md      # never-break flows → tier C smoke, every run
├── known-issues.md        # every confirmed bug, forever (each tagged with Alan:)
├── accepted-behaviours.md # looks like a bug, is intended → don't report
├── regression-log.md      # one line per run; drives tier D rotation
├── environment.md         # env manifest: targets, dependency reality, access
├── metrics.md             # one metrics row per run; `escaped` fed by postmortems
├── contracts/             # public-contract snapshots for the breaking-change diff
├── evidence/              # raw request/response per case — NEVER committed
├── reports/               # run reports: YYYY-MM-DD-<feature>.md (never at .qa root)
└── suites/
    └── <feature>.md       # case lists, each with a unique ID prefix (references/case-list.md)
```

**Scaling rules — cheap now, expensive later:**

- **`README.md` is the index, not documentation.** One line per file, plus the
  suite table: suite → prefix → area covered → case count → last run + verdict,
  and a short "uncovered areas" list (the tier D candidate pool). Update it in
  Phase 6; an index that drifts is worse than none. Flat files + a thin index
  beat deep folder hierarchies — this folder's main consumer greps.
- **Reports never live at the `.qa/` root** — always `reports/YYYY-MM-DD-<feature>.md`.
  One report per run lands here; the root must stay six core files.
- **Case IDs are suite-prefixed** (see `references/case-list.md`) so
  cross-references stay unambiguous as suites multiply.
- **Every `known-issues.md` entry carries an `Alan:` (area) tag.** The file grows
  forever by design; when it passes ~30–40 entries, split it by area into
  `known-issues/<area>.md` + an index — the tags make that split mechanical
  instead of archaeological.

**Version control:** `.qa/` is team memory — it belongs in git, like docs and
tests. The one exception is `evidence/`: raw exchanges can contain tokens and
PII, so add `.qa/evidence/` to `.gitignore` the first time you create it. And
no `.qa` file ever holds a credential — `environment.md` says *where* secrets
live, never what they are.

If `.qa/` doesn't exist, offer to create it on the first run: propose the files,
seed `critical-flows.md` from what the codebase obviously does (auth, payment,
the main business action), and let the user correct it. Don't create it silently
in someone's repo — ask once.

Keep these files short and factual. They are working memory, not documentation:
prune entries that stop being true.

**Suites grow AND shrink.** Tag each case in a suite as `core` (re-run every
time: bug-reproducers, money/permission/state cases, one representative per
equivalence class) or `swept` (passed clean twice on unchanged code → rotation
pool, re-run only when its area is tier A/B again or comes up in tier D). Merge
near-duplicates. The always-run core must stay executable in one sitting;
history lives in the file, not in re-execution.

---

## `.qa/critical-flows.md`

The flows that must never break. Smoke-tested every run (tier C) regardless of
what the diff touched.

```markdown
# Kritik akışlar

## CF-1 — API key ile kimlik doğrulama
- Adımlar: geçerli key ile GET /v1/... → 200; geçersiz key → 401
- Neden kritik: tüm entegrasyonlar buna bağlı
- Son doğrulama: 2026-08-20 ✅

## CF-2 — Sipariş oluşturma
- Adımlar: sepet → POST /v1/orders → DB'de kayıt + stok düşümü
- Neden kritik: gelir akışı
- Son doğrulama: 2026-08-20 ✅
```

## `.qa/known-issues.md`

Every confirmed bug, forever. A bug found once is the cheapest bug to find
twice — before designing a matrix, check whether any of these could recur in the
area you're testing.

```markdown
# Bilinen hatalar

## BUG-014 — Kupon eşzamanlı iki istekte iki kez uygulanıyor
- Alan: kupon / sipariş oluşturma
- Bulundu: 2026-08-20 | Severity: S1 | Durum: düzeltildi (commit abc1234)
- Kök neden: kupon kullanım sayacında kilit yok
- Kalıcı test: `tests/test_coupon.py::test_concurrent_redeem`
- Tekrarlama riski: kupon/kota/stok sayacı içeren her yeni özellik

## BUG-015 — Silinen kayıt export'ta görünüyor
- Bulundu: 2026-08-18 | Severity: S2 | Durum: açık
- Not: liste sorgusunda soft-delete filtresi var, export sorgusunda yok
```

## `.qa/accepted-behaviours.md`

Things that look like bugs but are intended. This file is what stops the report
from crying wolf on the same three items every release.

```markdown
# Kabul edilmiş davranışlar (bug değil)

- 404 yerine 403 dönmüyoruz: varlık sızıntısını engellemek için bilinçli tercih.
  (Karar: 2026-07-02, Ünal)
- Tarihler UTC saklanıp UI'da çevriliyor; API her zaman UTC döner. Kasıtlı.
- Boş isim alanı kabul ediliyor: eski entegrasyonlar bozulmasın diye.
```

## `.qa/regression-log.md`

One line per run. Gives you the rotation for tier D, and a history the user can
skim to see what has and hasn't been swept lately.

```markdown
# Test koşum günlüğü

| Tarih | Kapsam (tier A) | Rotasyon (tier D) | Senaryo | Bulgu | Karar |
|-------|-----------------|-------------------|---------|-------|-------|
| 2026-08-20 | kupon indirimi | webhook işleme | 47 | 1×S1, 2×S3 | NO-GO → düzeltildi → GO |
| 2026-08-14 | fatura PDF | kullanıcı davetleri | 38 | 1×S2 | GO WITH RISK |
```

## `.qa/environment.md`

The persistent form of Phase 0's environment capability inventory. Read it at
the start of every run and verify only what might have changed; update it in
Phase 6 with what the run taught you. Never a credential in here — only where
credentials live.

```markdown
# Ortam manifesti

## Hedef ortamlar
| Ortam | Erişim | Prod mu? | Not |
|-------|--------|----------|-----|
| lokal | http://localhost:<port> | hayır | seed: <projenin seed komutu> |
| staging | https://staging.example.com | hayır | test kullanıcıları: secret manager `qa/staging` |
| prod | — | **EVET — test edilmez** | sadece Phase 6.5 read-only smoke, istek üzerine |

## Erişim ve kimlik doğrulama (test playbook'u)
<!-- Yüzey başına: hangi yöntem koruyor + credential NASIL edinilir (script/seed/
     env adı/vault yolu). Secret'ın kendisi ASLA buraya yazılmaz. -->
| Yüzey | Yöntem | Credential edinme | Not |
|-------|--------|-------------------|-----|
| <kullanıcı API'si> | Bearer JWT | <token script'i / login akışı> | test kullanıcısı: <kim sağlar / vault yolu> |
| <admin yüzeyi> | Basic auth | env: <DEĞİŞKEN_ADI> | |
| <partner API'si> | API token | <seed adımı / üretme komutu> | scope: <...> |
| <webhook'lar> | imza / basic + IP filtresi | <nasıl taklit edilir> | |

## Dış bağımlılık gerçekliği
| Bağımlılık | lokal | staging | Not |
|------------|-------|---------|-----|
| kimlik sağlayıcı | gerçek | gerçek | |
| ödeme gateway'i | mock | sandbox | gerçek para asla |
| e-posta | yok | sandbox | gerçek adrese gönderim yasak |
| harici RPC servisi | YOK | gerçek | lokalde bu case'ler BLOCKED (ortam) |

## Fixture envanteri
<!-- Bağımlılığın gerçek olması yetmez: case'in ihtiyaç duyduğu tohum kayıt yoksa
     case koşulamaz ve bu, ortam eksiğinden daha sık rastlanan engeldir. Varlık
     başına ne VAR ne YOK yazılır; eksik olan tasarım anında BLOCKED (fixture)
     olur, koşum ortasında keşfedilmez. -->
| Varlık | Ortamda var olan | Eksik | Nasıl yaratılır |
|--------|------------------|-------|-----------------|
| <hesap / kiracı> | <2 adet; biri boş> | <ikinci rolde üye kullanıcı> | <seed komutu / API çağrısı> |
| <bakiye / kota> | <yalnızca sıfır bakiye> | <pozitif bakiyeli hesap> | <script> |
| <durum makinesi kaydı> | <draft, paid> | <expired, refunded> | <nasıl o duruma sürülür> |

## Bilinen ortam tuzakları
- <ör. lokalde saat dilimi UTC, staging'de değil>
```

## `.qa/metrics.md`

One row per run, appended in Phase 6. This is the skill's own scorecard — the
`escaped` column is filled retroactively by the postmortem loop and is the most
honest number in the table. Trends matter more than any single row: fake FAILs
and false alarms trending down means the process rules are working; `escaped`
staying at zero is the only claim that ages well.

```markdown
# Koşum metrikleri

| Tarih | Feature | Seviye | Case | Bulgu (S1/S2/S3/S4) | Bulgu/case | Yanlış alarm (Ph3 elenen) | Sahte FAIL (harness) | BLOCKED | Kaçan (escaped) | Maliyet (~) |
|-------|---------|--------|------|---------------------|------------|---------------------------|----------------------|---------|-----------------|-------------|
| 2026-08-20 | kupon indirimi | L3 | 120 | 0/1/2/0 | 0.025 | 1 | 3 | 1 | 0 | ~3 saat |
```

## `.qa/contracts/` and `.qa/evidence/`

- **contracts/** — one snapshot per public contract (OpenAPI JSON, schema dump,
  proto). Phase 0 diffs the live contract against it; breaking changes become
  tier A cases. Refresh only in Phase 6, after the verdict.
- **evidence/** — `<YYYY-MM-DD>-<feature>/<case-id>.*` raw exchanges for FAILs,
  reproduced findings and sampled PASSes. Git-ignored, pruned only with user
  consent.

---

## How to use it in a run

- **Phase 0:** read all four, plus any existing `suites/<feature>.md` for the area
  you're about to test. Critical flows become tier C cases; known issues
  become candidate recurrence cases; accepted behaviours become a do-not-report
  list; the log tells you which area to rotate into tier D. `environment.md`
  replaces rediscovering the environment; the `contracts/` diff seeds
  breaking-change cases.
- **Phase 1:** extend the existing case list rather than writing a new one.
- **Phase 3:** before promoting a finding, check it against
  `accepted-behaviours.md`.
- **Phase 6:** append the new bugs, log the run in both `regression-log.md` and the
  suite's own `## Koşum geçmişi`, add any new critical flow, record anything the
  user just declared intended.

One more compounding move worth suggesting to the user: every regression test
written during a run should be committed and wired into CI. The `.qa/` memory
keeps *you* from repeating work; CI keeps the *codebase* from regressing between
runs. Together they're the difference between a QA session and a QA practice.
