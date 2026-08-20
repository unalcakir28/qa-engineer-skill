# qa-engineer — Changelog

Semver. Her onaylı değişiklik buraya, hangi koşumun retrospektifinden geldiğiyle
birlikte yazılır. Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.5.0] — 2026-08-20

Motivasyon: kullanıcı direktifi — retro notlarındaki proje-bağımlı içerik skill
reposunda durmamalı.

### Changed

- **Retro artık öneri defteri, koşum günlüğü değil (Phase 7):** girdi yalnızca
  tarih + seviye + yüzey tipiyle anılır; içerik genelleştirilmiş dersler +
  öneriler. Genelleşemeyen ders projenin `.qa/`'sına gider. Provenance
  istisnası KALDIRILDI — günlükler de artık proje/ticket/domain terimi içeremez.
- Mevcut `RETROSPECTIVES.md` ve `CHANGELOG.md` girdileri bu kurala göre
  anonimleştirildi.

## [1.4.0] — 2026-08-20

Motivasyon: kullanıcı direktifi — "her test et dediğimde nasıl test edeceğini
yeniden keşfetmesin; çıkarabildiğini projeden çıkarsın, çıkaramadığını bir kez
sorup proje bazlı kaydetsin, değişince güncellesin."

### Added

- **Erişim playbook'u (Phase 0):** `environment.md`'ye kimlik doğrulama bölümü —
  yüzey başına yöntem + credential edinme yolu (script/seed/env adı/vault yolu;
  secret asla yazılmaz). Katı sıra: (1) dökümante edilmişse aynen kullan,
  yeniden keşfetme; (2) değilse projeden çıkar (guard'lar, auth config, mevcut
  script'ler, projenin kendi testleri); (3) çıkarılamıyorsa kullanıcıya BİR KEZ
  sor ve cevabı manifeste yaz. Dökümante edilmiş bir bilgiyi ikinci kez sormak
  süreç hatasıdır → retrospektife loglanır. Yöntem değişirse/bozulursa/yenisi
  eklenirse Phase 6'da manifest güncellenir.
- **Git release protokolü:** skill dizini git reposu oldu
  (`unalcakir28/qa-engineer-skill`, private). Onaylı her sürüm artışı =
  commit + `vX.Y.Z` tag + push (bu repo için kalıcı izin, 2026-08-20; proje
  repolarını kapsamaz). Retro girdileri tag'siz commit'lenir.

## [1.3.0] — 2026-08-20

Motivasyon: kullanıcıyla iyileştirme oturumu — 8 öneri tartışıldı, tamamı
onaylandı.

### Added

- **Kaçan bug postmortem döngüsü:** prod'a kaçan her bug için atıf (hangi koşum,
  hangi kategori) → miss teşhisi (design gap / false PASS / parked / out of
  scope) → regresyon case + known-issues + metrics `escaped` artışı →
  genelleşiyorsa otomatik P1 önerisi.
- **`.qa/environment.md` ortam manifesti:** Phase 0 envanteri artık kalıcı —
  okunur, doğrulanır, Phase 6'da güncellenir; credential asla yazılmaz.
- **`.qa/metrics.md` etkinlik metrikleri:** koşum başına bulgu/case, yanlış
  alarm, sahte FAIL, BLOCKED, kaçan bug, maliyet — "gelişiyor muyuz"un sayısal
  cevabı.
- **`.qa/evidence/` kanıt arşivi:** FAIL/repro/örneklenmiş PASS'lerin ham
  request/response'u case ID başına dosyada; **asla commit edilmez**
  (`.gitignore`'a eklenir), budama sadece kullanıcı onayıyla.
- **`.qa/contracts/` contract-diff:** public sözleşme snapshot'ı her koşumda
  diff'lenir; breaking change otomatik tier A case + bulgu adayı olur.
- **`references/test-data.md`:** fixture izolasyonunun tarifi — prefix şeması,
  izolasyon seviyeleri, idempotent seed, sentetik değer geçerliliği (validasyonu
  değil business kuralını test et), zaman sınırları, temizlik, secrets kuralları.
- **PR modu:** tier A = PR diff'i; standart rapora ek yoğunlaştırılmış PR-yorumu
  çıktısı; PR'a gönderim yalnızca açık onayla.
- **Sentinel modu:** zamanlanmış/CI koşumları için tanım — L2 report-only,
  tier C smoke + tier D rotasyon + son koşumdan beri değişen tier A; gözetimsiz
  asla kod düzeltmez, veri budamaz, dışarı göndermez.
- **`.qa/` versiyon kontrol kuralı (qa-memory.md):** `.qa/` commit edilir (takım
  hafızası), `.qa/evidence/` ignore edilir, hiçbir `.qa` dosyası secret tutmaz.

## [1.2.0] — 2026-08-20

Motivasyon: kullanıcı direktifi — skill her projede (frontend/backend, her
stack) değişmeden çalışmalı.

### Added

- **Proje-bağımsızlık kuralı (Phase 7):** `SKILL.md` ve `references/` asla
  proje/ticket/endpoint/framework-dekoratörü/domain kavramı içermez; her ders
  yalnızca genelleştirilmiş kalıp olarak kabul edilir (turnusol: "bu cümle başka
  bir repoda da aynen doğru mu?"). Genelleşemeyen bilgi projenin `.qa/`
  hafızasına gider. Araç isimleri yalnızca ekosistem-başına seçim kurallı menü
  olabilir. Günlükler (CHANGELOG/RETROSPECTIVES) provenance istisnasıdır.

### Denetim notu

- v1.2.0 itibarıyla SKILL.md + references/ tarandı: proje-spesifik içerik yok.
  `automation-toolbox.md` (ekosistem menüsü) ve `oracles.md` (emsal API örneği)
  kurala uygun bulundu.

## [1.1.0] — 2026-08-20

Motivasyon: ilk gerçek saha koşumunun retrospektifi (2026-08-20, L3, backend
API, 120 case, GO). Bkz. `RETROSPECTIVES.md` → 2026-08-20.

### Added

- **Phase 7 — Skill retrospective:** her koşumun sonunda skill kendini
  değerlendirir, `RETROSPECTIVES.md`'ye girdi yazar, iyileştirme önerilerini
  kullanıcıya sunar; skill dosyaları yalnızca onayla değişir.
- **Versiyonlama:** front-matter `version` alanı + bu CHANGELOG; PATCH/MINOR/MAJOR
  kuralları Phase 7'de tanımlı.
- **Fixture izolasyonu kuralı (Phase 2):** state mutasyonu yapan her case kendi
  fixture'ını kurar veya geri alır; paralel ajanlar asla veri paylaşmaz, shared
  state'e dokunan grup tek başına koşar. (Koşumda 3 sahte FAIL üretmişti.)
- **"Önce kendi harness'ını suçla" refleksi (Phase 2):** beklenmedik toplu
  FAIL'de response body loglanır, validasyon reddi ile business reddi ayrılır.
  (Koşumda format validasyonundan geçmeyen sentetik bir değer bir case grubunun
  tamamına sahte 400 döndürmüştü.)
- **Subagent kanıt sözleşmesi + PASS örneklemesi (Phase 2 paralel bölümü):** ham
  request/response zorunlu; yüksek riskli kategorilerden PASS örneklemi ana
  session'da yeniden koşulur; tutmayan örneklem o ajanın tüm grubunu yeniden
  doğrulatır. (Koşumda TC-099 FAIL'i subagent'ın kendi test hatası çıkmıştı.)
- **Ortam yetenek envanteri (Phase 0):** dış bağımlılıklar tasarım anında
  gerçek/mock/yok olarak işaretlenir; yok olanların case'leri baştan
  `BLOCKED (ortam)` alır. (Koşumda bir dış RPC bağımlılığının yokluğu koşum
  sırasında keşfedilmişti.)
- **Seviye sorusuna maliyet etiketi (Kickoff):** her seviye seçeneği tahmini
  case sayısı + süre/token maliyetiyle sunulur.
- **BLOCKED borç takibi (Phase 4 + 6):** raporda "başka ortamda koşulacaklar"
  listesi; `regression-log.md`'ye sonraki koşumun girdisi olarak devredilir.
- **Suite budama stratejisi (Phase 6):** case'ler core/swept olarak etiketlenir;
  her koşumda koşulan çekirdek küçük tutulur, temiz geçenler rotasyon havuzuna
  iner, benzer case'ler birleştirilir.

### Notlar

- Severity rubrik'i önerisi uygulanmadı: rubrik zaten
  `references/reporting.md`'de mevcuttu (S1–S4 + Risk + Question).

## [1.0.0] — 2026-08-20

- İlk sürüm: kickoff (seviye + fix/report), Phase 0–6.5, 20 satırlık kategori
  kataloğu, tier A–D kapsam modeli, non-negotiable kanıt kuralları, model
  politikası (tasarım/karar Opus, koşum Sonnet), `.qa/` proje hafızası,
  references/ altında 10 yardımcı doküman.
