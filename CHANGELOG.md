# qa-engineer — Changelog

Semver. Her onaylı değişiklik buraya, hangi koşumun retrospektifinden geldiğiyle
birlikte yazılır. Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.9.0] — 2026-09-22

Motivasyon: 2026-09-22 L3 koşumunun retrospektifi (backend servis katmanı,
eşzamanlılık/kilit düzeltmesi; projenin mevcut mock'lu süiti tamamen yeşilken
değişikliğin ana iddiası hakkında sıfır bilgi taşıyordu).

### Added

- **Non-negotiable #9 — test, iddianın yaşadığı seviyede koşmalı.** Bir
  değişiklik eşzamanlılık, kilitleme, izolasyon, transaction, atomiklik veya
  sıralama hakkında bir iddiada bulunuyorsa, mevcut test süiti ne kadar yeşil
  olursa olsun o iddianın kapsamı değildir: test double'ları kilit tutmaz,
  izolasyon seviyesi uygulamaz, commit sırası üretmez. Böyle bir değişiklik en az
  bir case'i gerçek bağımlılığa karşı (gerçek veritabanı, gerçek broker, gerçek
  paralel süreçler) koşmalı; koşulamıyorsa rapor iddianın test edilmediğini
  bu kelimelerle yazar.
- **Phase 3 — zamanlama bulgusu, zararsız sıralamalar ayıklanmadan bir sayı
  değildir.** Paralel bir koşumda her tur "kötü" değildir; koşulun zaten
  herkesten sonra devreye girdiği turlar beklenen davranıştır. Bir turun bulguya
  sayılması için, bir aktörün yeni durumu gerçekten gördüğünün gösterilmesi
  gerekir. Bir bulgu ilk ölçümde 25'te 13 çıkmıştı; ayıklama sonrası gerçek rakam
  30'da 3 oldu — aynı hata, onda bir iddia.

### Changed

- **Non-negotiable #3 (kırmızı-yeşil) eşzamanlılık için genişletildi.** Yalnızca
  eşzamanlılık, zamanlama veya yük altında ortaya çıkan bir hatada yeşil sonuç,
  aynı harness'ın düzeltilmemiş kodda kırmızı verdiği gösterilene kadar bilgi
  taşımaz — aksi halde yeşil ile "senaryo hiç gerçekleşmedi" ayırt edilemez.
- **`references/techniques.md` §11 (A/B diferansiyel)**, tekniğin ikinci
  işlevini açık yazıyor: atıf kadar **harness geçerliliği**. Eşzamanlılık
  iddialarında temel build'in kırmızısı, yeni build'in yeşiline inanmanın
  lisansıdır.
- **Katalog satırı 9 (eşzamanlılık & idempotency)** artık en az bir case'in
  gerçek bağımlılığa karşı koşulmasını istiyor ve #9'a atıf veriyor.

## [1.8.0] — 2026-09-16

Motivasyon: 2026-09-16 L3 koşumunun retrospektifi (backend API, güvenlik sınırı
düzeltmesi, koşumu yapan kişi aynı zamanda değişikliği yazan kişiydi).

### Added

- **`references/techniques.md` §11 — diferansiyel (A/B) yürütme.** Değişiklik
  öncesi build, mevcut build'in yanında **aynı** veri deposuna karşı ayağa
  kaldırılır ve atıf gerektiren her prob iki kez koşulur. "Bu bulguyu ben mi
  yarattım" sorusu tartışma olmaktan çıkıp ölçüme dönüşür; dokunulmaması gereken
  yüzeylerin bayt bayt karşılaştırması tasarlanmamış bir regresyon taraması
  verir; temel build zaten düzeltme öncesi kod olduğu için kırmızı-yeşil kanıtı
  bedava gelir. Temelin geçerliliğini (hangi commit'ten / hangi zamandan) ilk
  sonuçtan **önce** kanıtlama zorunluluğu kuralın parçası — aksi halde tüm
  "önceden var" etiketleri dayanaksız kalır.
- **Phase 0 — fixture envanteri.** Bağımlılık envanterinin ikizi: varlık başına
  ortamda hangi tohum kaydın olduğu ve hangisinin olmadığı. Eksik olan case
  tasarım anında `BLOCKED (fixture)` olur. Kalıcı yeri `.qa/environment.md`
  (şablon `qa-memory.md`'de). Gerekçe: o koşumdaki BLOCKED'ların neredeyse
  tamamı ortam değil **veri** eksikliğiydi ve hepsi koşum ortasında keşfedildi.
- **Phase 0 — "temel build koşulabilir mi" sorusu.** A/B kararı Phase 0'da
  verilir, çünkü case listesini (her atıflı prob iki kez) ve kararı birden
  etkiler; uygulanamıyorsa hangi sebeple olduğu yazılır.

### Changed

- **Karar kurallarına atıf boyutu (`release-gate.md`).** Tablo "açık S1/S2"
  diyordu ama *kimin* olduğunu sormuyordu. Düz uygulandığında, ciddi bir açığı
  kapatan değişiklik yanından geçtiği ilgisiz bir açık yüzünden bloke oluyor ve
  bu **ikisini birden** üretimde bırakıyor. Artık karar değişikliğe
  **atfedilebilen** bulgular üzerinden hesaplanır: A/B ile önceden var olduğu
  doğrulanan bulgu `NO-GO` üretmez, kararı `GO WITH RISK`'te tutar ve kendi
  ticket'ı olur. İki istisna onu yeniden değişikliğin hanesine yazar —
  değişikliğin görevi onu düzeltmekse, ya da değişiklik onu daha erişilebilir /
  daha ağır hale getiriyorsa. Yalnızca yeni build'de görünen bulgu tam ağırlıkla
  regresyondur.
- **Phase 3 — "önceden var" bir ölçümdür, sezgi değil.** Bu etiket artık kararı
  değiştirdiği için bulgunun kendisiyle aynı kanıt standardına tabi: temel build
  varsa prob orada da koşulur ve iki çıktı da eklenir; yoksa etiket *muhtemelen
  önceden var* olarak yazılır.
- **Non-negotiable #6'ya tasarım bağımsızlığı eklendi.** "Kendi düzeltmeni taze
  gözle doğrula" yürütmeyi kapsıyordu, tasarımı değil: değişikliği yazan kişi
  case listesine kendi kör noktalarını da miras bırakır — hiç düşünmediği şey
  için case tasarlayamaz. Kendi yazdığı yeni kodun case'leri, onu yazan zihinden
  başka bir şey tarafından tasarlanmalı veya gözden geçirilmeli.
- **`test-data.md` — "değeri oku, ismi değil".** Sentetik değer kuralının format
  kontrolünden kaçan hâli: enum/scope/durum kodu gibi tanımı başka yerde olan
  her şeyin **değeri** tanımdan okunur, koddaki tanımlayıcı adı yeniden
  yazılmaz. Adı `READ_ONLY` olup değeri `read:only` olan bir üye, tıpkı bir yazım
  hatası gibi validasyondan döner ve gerçek bug'dan ayırt edilemez. O koşumda
  iki sahte FAIL'in kaynağı buydu.

## [1.7.0] — 2026-09-14

Motivasyon: koşum retrospektifi değil — skill'in kendisinin gözden geçirilmesi
(kullanıcı isteğiyle, yapı ve maliyet denetimi).

### Added

- **`references/cli-tool.md`:** üçüncü yüzey checklist'i — komut satırı araçları
  ve ikili dosyalar. Sözleşme çıkış kodu + stdout/stderr + diske yapılan iş
  olarak tanımlanır; argüman ayrıştırma, dosya sistemi düşman girdileri
  (symlink döngüsü, sparse, NFC/NFD, izin hataları, ölü mount), sinyal ve
  iptal, akış/pipe/TTY davranışı, config önceliği, ayrıcalık sınırları,
  çapraz platform ve paketleme bölümleri. Daha önce yalnızca `backend-api` ve
  `web-frontend` vardı — CLI yüzeyi tamamen kapsam dışıydı.
- **Yüzey kategoriyi belirler kuralı (Phase 1):** bir kategori geçerli değilse
  gerekçesiyle bir kez yazılır; sessizce atlanmaz, boş case de üretilmez.

### Changed

- **Phase 6.5 yanlış yerdeydi:** dosyada Phase 6'dan *önce* geliyordu. Phase
  6'dan sonraya alındı ve `release-gate.md`'ye işaret eden üç satıra indirildi.
- **Soğuk bölümler `references/`'a taşındı** — SKILL.md 7.773 → 6.851 kelime
  (her tetiklenmede yüklenen maliyet; ~1.450 kelime taşındı, yerine ~250 kelime
  işaretçi ve ~130 kelime yeni kural girdi). Taşınanlar: PR/Sentinel koşum modları ve
  paralel koşum kuralları → `run-modes.md`; kaçan bug postmortem döngüsü →
  `postmortem.md`; retrospektif girdi formatı, genelleme testi, semver ve
  release yordamı → `skill-maintenance.md`. Hepsinin yerinde tek satırlık
  işaretçi var; hiçbiri normal bir koşumun sıcak yolunda değildi.
- **`description` 1008 → 797 karakter.** 1024 sınırına 16 karakter kalmıştı;
  eş anlamlı Türkçe tetikleyiciler ("test yap", "hata bulmaya calis",
  "canliya cikmadan once kontrol et") budandı, "CLI command" eklendi.

## [1.6.0] — 2026-08-20

Motivasyon: kullanıcıyla `.qa/` ölçekleme değerlendirmesi — klasör büyüdükçe
kaybolmadan arananı bulma.

### Added

- **`.qa/README.md` indeksi:** dosya haritası + suite tablosu (prefix, kapsanan
  alan, case sayısı, son koşum, karar) + kapsanmamış alanlar (tier D havuzu);
  Phase 6'da güncel tutulur. İlke: düz dosyalar + ince indeks > derin klasör
  hiyerarşisi — bu klasörün ana tüketicisi grep'ler.
- **Suite-prefix'li case ID'leri:** her suite dosya başında benzersiz kısa slug
  tanımlar (`KPN-001`), çıplak `TC-` yasak — suite çoğaldıkça çapraz referans
  belirsizliğini önler. Şablon ve örnekler güncellendi.
- **`.qa/reports/` klasörü:** raporlar `YYYY-MM-DD-<feature>.md` adıyla buraya;
  `.qa` kökü 6 çekirdek dosyada sabit kalır.
- **known-issues `Alan:` etiketi:** kayıt sonsuza dek büyür; ~30-40 kayıtta alan
  bazlı bölünme etiketler sayesinde mekanik olur.

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
