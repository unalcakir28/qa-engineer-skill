# Reporting: severity rubric, template, quality bar

## Severity rubric

| Severity | Meaning | Examples |
|---|---|---|
| **S1 – Critical** | Data loss/corruption, security or tenancy breach, money wrong, feature unusable for everyone, no workaround | Another tenant's records readable; payment charged twice; stock goes negative; totals wrong; auth bypass; unrecoverable state |
| **S2 – High** | Core scenario broken or silently wrong for a real subset of users; ugly workaround only | Validation missing so bad data persists; illegal state transition accepted; list leaks soft-deleted rows; race creates duplicates; 500 on a normal flow |
| **S3 – Medium** | Edge case wrong, poor error handling, misleading message, wrong-but-recoverable behaviour | Unhelpful/raw error text; boundary off-by-one on a rare limit; pagination duplicates a row; timezone shifts a display date |
| **S4 – Low** | Cosmetic, wording, minor UX, inconsistency with no functional impact | Label truncation; inconsistent date format; missing loading indicator |
| **Risk** | Not reproduced (yet) but code/behaviour suggests a real problem | Unbounded query with no index; `catch {}` swallowing errors; missing idempotency key |
| **Question** | Requirement is ambiguous — behaviour may be intended | "Should an expired coupon on a pending order stay valid?" |

Silent-and-wrong beats loud-and-broken in severity: a wrong number nobody
notices is worse than an error page.

## Report template

Use this structure. Keep it scannable — the worst finding goes at the top.

```markdown
# Test Raporu — <feature> — <date>

## KARAR: <GO | GO WITH RISK | NO-GO>
> Gerekçe: <the rule that produced it, e.g. "açık 1 adet S1 (#3)">
> <If NO-GO: what must be fixed before this can ship.>

## Özet
- Seviye: <L1 smoke | L2 standart | L3 sürüm kapısı | odaklı> — <kullanıcı seçti / varsayılan>
- Kapsam: <branch/commit, environment, tier planı (A: … / B: … / C: … / D: …)>
- Sonuç: <N> senaryo koşuldu — <N> PASS / <N> FAIL / <N> BLOCKED / <N> NOT RUN
- Bulgular: <N>×S1, <N>×S2, <N>×S3, <N>×S4, <N> risk, <N> açık soru
- En kritik bulgu: <one sentence>

## Bulgular (severity sırasına göre)

### [S1] <short title>  ·  `TC-021`
- **Nerede:** <endpoint / screen / file:line>
- **Tekrar üretme:**
  1. <exact step, with the exact request/input>
  2. ...
- **Beklenen:** <...> — *dayanak:* <requirement / schema / constraint / doc>
- **Gerçekleşen:** <...>
- **Kanıt:** <real response body / log line / DB row / screenshot path>
- **Doğrulama:** temiz state'te <N> kez tekrar üretildi; elenen alternatif
  açıklamalar: <stale build / hatalı test verisi / yanlış istek / kasıtlı davranış>
- **Muhtemel sebep:** <file:line + one-line diagnosis, if known>
- **Etki:** <who is affected and how badly>
- **Yeni mi:** <bu değişiklikle geldi | önceden de vardı>

### [S2] ...

## Riskler ve gözlemler
- <not reproduced, but worth attention — with reasoning>

## Açık sorular
- <ambiguous requirement — needs a product decision>

## Kapsam özeti (kategori bazında)

Kataloğun her satırı burada görünmeli; sıfır olan satırın gerekçesi yazılmalı.

| # | Kategori | Senaryo | PASS | FAIL | NOT RUN | Not / gerekçe |
|---|----------|---------|------|------|---------|----------------|
| 1 | Happy path | 5 | 5 | 0 | 0 | |
| 2 | Fonksiyonel mekanik | 8 | 7 | 1 | 0 | |
| 3 | Negatif / validasyon | 11 | 9 | 2 | 0 | |
| 4 | Sınır değer | 9 | 8 | 1 | 0 | |
| 5 | Denklik sınıfı | ... | | | | |
| 6 | Kombinasyon (karar tablosu / pairwise) | ... | | | | |
| 7 | Yetki / kiracılık | ... | | | | |
| 8 | Durum & sıra | ... | | | | |
| 9 | Eşzamanlılık & idempotency | ... | | | | |
| 10 | Veri bütünlüğü | ... | | | | |
| 11 | Kenar veri | ... | | | | |
| 12 | Hata yönetimi & dayanıklılık | ... | | | | |
| 13 | Güvenlik | ... | | | | |
| 14 | Regresyon & entegrasyon | ... | | | | |
| 15 | Keşifsel / hata tahmini | ... | | | | |
| 16 | Performans | 0 | | | | tek kayıtlı dev DB — anlamlı ölçüm yok |
| 17 | Uyumluluk / responsive / a11y | ... | | | | |
| 18 | Yerelleştirme & formatlar | ... | | | | |
| 19 | Config / migration / deploy | ... | | | | |
| 20 | Gözlemlenebilirlik | ... | | | | |

## Case listesi

Tam liste dosyada: `.qa/suites/<feature>.md` (veya `test-cases-...md`).
Raporda ya dosyaya link ver ya da tabloyu buraya kopyala.

| ID | Tier | Kategori | Senaryo | Beklenen | Gerçekleşen | Durum |
|----|------|----------|---------|----------|-------------|-------|
| TC-001 | A | Happy | ... | ... | ... | PASS |
| TC-014 | A | Sınır | ... | ... | ... | FAIL (S2) |
| TC-041 | C | Kritik akış smoke | ... | ... | ... | PASS |

## Kapsam dışı bırakılanlar
- <seviye gereği atlanan kategoriler — hangileri, neden>
- <bilinçli deprioritize edilenler — dürüst ol>

## Eklenen otomatik testler
- `path/to/test_file.py::test_name` — <what it guards> (was failing before fix X)

## Prod öncesi kontrol listesi
| Alan | Durum | Not |
|------|-------|-----|
| Migration & veri (geri alınabilir mi, index'ler) | OK / RISK / N/A | |
| Geriye uyumluluk (eski istemciler, kuyruk mesajları) | | |
| Config & feature flag (kapalı yol dahil) | | |
| Güvenlik & gizlilik (yetki, log'da sır/PII, rate limit) | | |
| İşletilebilirlik (log, metrik, geri dönüş planı) | | |
| Temizlik (test verisi, CI'a eklenen testler) | | |

## Bir insanın hâlâ kontrol etmesi gerekenler
- <real payment / e-mail deliverability / görsel-UX / gerçek yük / erişemediğim hesap>

## Önerilen düzeltme sırası
1. [S1] <bulgu> — <tek satır çözüm fikri>
2. [S2] ...

> Rapor-only modda rapor burada biter: "Bu düzeltmeleri yapmamı ister misin?"

## Yapılan düzeltmeler *(yalnızca düzeltme istendiyse)*
| # | Bulgu | Düzeltme | Dosya | Doğrulama |
|---|-------|----------|-------|-----------|
```

## Quality bar for a finding

A finding is only useful if someone else can act on it. Before writing one down:

- **Verified:** it survived the Phase 3 refutation pass — reproduced from a clean
  state, alternative explanations ruled out, checked against
  `.qa/accepted-behaviours.md`. An unverified item is a Risk, not a bug. When
  there is no human tester downstream, a false alarm is more expensive than a
  missed S4: it trains the reader to skim.
- **Reproducible:** exact input, exact steps, minimal case. "Sometimes fails" is
  not a bug report — find the trigger, or file it as a Risk and say what you saw.
- **Evidence, not adjectives:** paste the actual response, log line or DB row.
  Never invent output you didn't observe.
- **One bug per entry:** don't bundle three problems into one item; don't split
  one root cause into eight items (group them and name the root cause once).
- **Expected clearly justified:** cite the requirement, doc, schema constraint
  or convention that makes the actual behaviour wrong. If nothing justifies it,
  it belongs in Açık sorular, not Bulgular.
- **No blame, no drama:** describe behaviour, not the developer.

## Honesty rules

- Verdicts reflect what you actually ran. `NOT RUN` is respectable; a fake
  `PASS` destroys the value of the whole report.
- If the environment blocked you, say which cases were blocked and why.
- If you found nothing, the report is still the deliverable: the case list with
  its statuses shows what was attacked, which is what makes "no bugs found"
  believable.
- State the depth you chose and what you skipped. Untested-and-declared is fine;
  untested-and-implied-tested is a lie the user will discover in production.
