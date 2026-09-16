# qa-engineer — Öneri defteri (koşum retrospektifleri)

Her koşumun sonunda (Phase 7) bir girdi. **Bu bir öneri defteridir, koşum
günlüğü değil:** koşum yalnızca tarih + seviye + yüzey tipiyle anılır; proje,
ticket, endpoint veya domain terimi buraya girmez (koşumun hikâyesi ilgili
projenin `.qa/` klasöründedir). Öneriler P1 (yanlış sonuca yol açtı) / P2
(ciddi zaman kaybı) / P3 (cila) olarak derecelendirilir. Tekrarlayan öneri
otomatik P1'e yükselir. Uygulanan öneriler `CHANGELOG.md`'ye taşınır ve burada
`UYGULANDI (vX.Y.Z)` olarak kapatılır.

---

## 2026-08-20 — L3, backend API

- **Case:** 120 · **Sonuç:** 117 PASS / 1 FAIL (S2, düzeltildi) / 1 BLOCKED /
  1 SKIP · **Karar:** GO · **Skill sürümü:** 1.0.0

**Genelleştirilmiş dersler:**

- Katalog satırı 8 (durum & sıra), "kapanmış kaydı aynı sebeple yeniden tetikle"
  case'iyle doğaçlamanın bulamayacağı bir S2 yakaladı; kırmızı-yeşil kuralı ve
  kendini-sertifikalandırma yasağı düzeltmenin kanıtını sağlamlaştırdı.
- State mutasyonu yapan senaryolar izole fixture olmadan sonraki case'lere sahte
  FAIL sızdırdı (3 adet).
- Format validasyonundan geçmeyen sentetik bir değer, bir case grubunun tamamını
  business kuralına ulaşmadan reddettirdi — neredeyse kod bug'ı olarak
  raporlanacaktı.
- Bir subagent FAIL'i, subagent'ın kendi hatalı isteğinden kaynaklandı; ham
  request/response olmadan ayırt edilemiyordu.
- Bir dış bağımlılığın hedef ortamda bulunmadığı koşum ortasında keşfedildi —
  ilgili case'lere boşa efor harcandı.

**Öneriler:**

- P1 — fixture izolasyonu kuralı → UYGULANDI (v1.1.0)
- P1 — harness-first şüphe refleksi → UYGULANDI (v1.1.0)
- P2 — subagent ham kanıt + PASS örneklemesi → UYGULANDI (v1.1.0)
- P2 — Phase 0 ortam yetenek envanteri → UYGULANDI (v1.1.0)
- P2 — seviye seçeneklerine maliyet etiketi → UYGULANDI (v1.1.0)
- P3 — BLOCKED borç takibi → UYGULANDI (v1.1.0)
- P3 — suite budama stratejisi → UYGULANDI (v1.1.0)
- P3 — severity rubrik'i önerisi KAPANDI: rubrik zaten
  `references/reporting.md`'de varmış; sorun keşfedilebilirlikti, Phase 4 zaten
  oraya yönlendiriyor.

**Açık öneri (onay bekliyor):** yok.

---

## 2026-08-21 · L3 · backend API (para akışı + zamanlanmış iş + escrow muhasebesi)

**Genelleştirilmiş dersler:**

- **Bir endpoint HTTP 2xx döndüğünde işin bitmiş olduğu varsayımı, en verimli
  sahte FAIL üreticisi.** Yazma isteği bir kuyruğa devrettiğinde yanıt, kalıcı
  duruma değil "kayıt alındı"ya işaret eder; hemen ardından yapılan bakiye/durum
  okuması işçiyle yarışır. Bu koşumda tek başına 3 phantom FAIL üretti ve bunlar
  ürün bugu olarak raporlanmaya bir adım kalmıştı. Kural adayı: *durum
  değiştiren asenkron bir akışta assert etmeden önce terminal duruma geçişi
  beklemek zorunlu; bekleme yoksa case tasarımı eksik sayılır.*
- **Para karşılaştırmasında dilin varsayılan sayı tipi kullanılamaz.** Ondalık
  bakiyeler üzerinde float çıkarması (`100000 - 99999.01`) kuruş seviyesinde
  yapay uyuşmazlık üretti ve "MISMATCH" olarak raporlanmaya hazırdı. Kural
  adayı: *para değişmezleri yalnızca ondalık/rasyonel tipte (ya da tamsayı kuruş)
  karşılaştırılır.*
- **Paylaşılan harness, tek bir yanlış yol ile birden çok ajanın kanıtını
  sessizce bozar.** Yanlış endpoint yolu 404 döndürüyordu; 404'ü "yetki reddi"
  sanan bir yetki case'i sahte PASS üretebilirdi. İki ajan bağımsız olarak
  yakaladı. Kural adayı: *paylaşılan harness'ın her endpoint yolu, ilk kullanımda
  "yol var mı" ayrımıyla doğrulanmalı (404 ≠ 401/403); yol düzeltmesi koşan tüm
  ajanlara duyurulmalı.*
- **Case'in kendi formülü de bir dayanak hatası olabilir.** Bir muhasebe
  değişmezini net/brüt karıştıran şekilde yazmıştım; ajan "FAIL" raporladı ve
  doğru teşhisi kendisi koydu (artık tam olarak iade edilen tutara eşitti).
  Sistem doğruydu, spec yanlıştı. Kural adayı zaten var (*basis'in kendisi
  kusurlu olabilir*), ama **sayısal değişmezler** için özel olarak zayıf: bir
  koruma değişmezi yazarken net akış ile brüt akışın ayrıştırılması gerekiyor.
- **Uzun bekleme gerektiren case'ler (cron/periyodik iş) ajan bloklar ve rapor
  gecikir.** İki ajan kendi arka plan beklemesinde durdu; sonuçları ancak
  "elindekini raporla" mesajıyla alındı. Bu arada aynı özelliğin deterministik
  bir eşdeğeri (kuyruğa doğrudan iş bırakmak, ya da tetikleyici koşulun
  DB'deki izini kontrol etmek) saniyeler sürüyordu. Kural adayı: *zamana bağlı
  bir mekanizma için önce deterministik eşdeğerini koş, gerçek zamanlayıcıyı
  yalnızca bir kez uçtan uca doğrulamak için bekle; bekleme case'lerini ayrı bir
  gruba topla ki rapor onlara takılmasın.*
- **Bir ajanın "FAIL"i, case'in beklentisi kodun belgelenmiş sözleşmesinden
  daha katı olduğu için de gelebilir.** Güvenlik davranışı doğruydu (kredi yok,
  gürültülü hata, DLQ alarmı); yalnızca benim beklediğim teşhis alanları
  dolmuyordu. Ph3 doğrulaması bunu bulguya dönüşmeden ayıkladı.
- **Paralel ajanların doğrudan DB mutasyonları, başka ajanın global değişmez
  taramasında "ihlal" olarak görünür.** Bir ajanın kasıtlı olarak bozduğu satır,
  başka ajanın bütünlük taramasını FAIL'e düşürdü; `referenceId` ön eki
  sayesinde izlenebildi. Kural adayı: *global değişmez taraması yapan case,
  ihlal bulduğunda satırı fixture ön ekiyle sahiplendirmeli; sahibi başka bir
  ajansa bu bir ürün bulgusu değildir.*

**Öneriler:**

- **P1 — Asenkron kabul (202/201-then-worker) kalıbı için zorunlu bekleme
  kuralı.** `SKILL.md` Phase 2'ye ve `references/backend-api.md`'ye: bir yazma
  isteği işi kuyruğa/işçiye devrediyorsa, assert öncesi terminal duruma geçişi
  beklemek case tasarımının parçasıdır. (Bu koşumda 3 phantom FAIL; bir sonraki
  koşumda tekrar ederse "harness-first şüphe" kuralı bunu yakalıyor ama maliyeti
  yüksek.)
- **P1 — Para değişmezlerinde ondalık aritmetiği zorunluluğu.** `SKILL.md`
  "Assert values, not vibes" maddesine bir cümle: para/oran karşılaştırmaları
  dilin float tipiyle yapılmaz. (Bu koşumda 1 phantom finding.)
- **P2 — Paylaşılan harness'ta endpoint yolu doğrulama refleksi.** `references/
  test-data.md` veya `backend-api.md`: yolun varlığını 404/401 ayrımıyla teyit
  et; paylaşılan harness düzeltmesini koşan ajanlara duyur.
- **P2 — Zaman-bağımlı case'ler için "deterministik eşdeğer önce" kuralı** ve
  bekleme case'lerinin ayrı gruba toplanması (`SKILL.md` paralelleştirme
  bölümü).
- **P3 — Global değişmez taramalarında ihlal sahipliği.** Paralel koşumda bulunan
  ihlal, fixture ön ekiyle sahiplendirilmeden bulgu sayılmaz.

**Açık öneri (onay bekliyor):** yukarıdaki P1/P1/P2/P2/P3 — kullanıcı onayı
bekleniyor. Onaylanırsa MINOR sürüm (yeni kurallar davranışı değiştiriyor):
v1.7.0.

---

## 2026-08-27 · L3 · backend API (görünürlük/bilgi-açığı sınırı)

**Skill'in yapısı neyi yakaladı:**

- **Kontrat taraması, iki review turunun kaçırdığı bulguyu buldu.** Feature iki
  ayrı kod-review turundan geçmişti; bulunan iki S3'ten birini (iç bayrağın
  istemci sözleşmesine sızması) yakalayan şey, keşif charter'ını "şemayı
  sistematik tara" biçiminde koşmak oldu: yayınlanan API şemasında hedef alanı
  taşıyan **tüm** uçları programatik olarak çıkarmak. Tek tek uç denemek bunu
  bulmuyordu; şema grafiği buluyordu.
- **Kardeş uç kıyası, ikinci S3'ün oracle'ı oldu.** "Aynı ailedeki üç uç bu
  girdiye 400 dönüyor, bu 500 dönüyor" — spec'te yazmayan bir kuralı
  anchor'layan en güçlü kanıt buydu.
- **"Harness'ından şüphe et" kuralı bir yanlış alarmı raporlanmadan önce kesti.**
  Yetki bypass'ı gibi görünen davranışın sebebi, testin kendi verdiği yükseltilmiş
  roldü. Guard kodunu okumak, IDOR raporlamaktan kurtardı.

**Süre/gürültü maliyeti:**

- **Derleme adımı, çalışan watch sürecini öldürdü.** Doğrulama turunun ortasında
  "hepsini kontrol et" komutunu koşmak, watch sürecinin okuduğu build çıktısını
  yeniden yazdı; uygulama modül bulamayıp çöktü ve tüm doğrulama istekleri
  bağlantı hatası döndü. Bir tur boşa gitti.
- **Ücret/bakiye ön koşulu, üç case'i ajan tarafında BLOCKED yaptı.** Yazma yolu
  case'leri paylaşılan fixture'a dokunmama kuralı yüzünden bloke oldu; ana
  oturumda ön koşul sağlanınca üçü de saniyeler içinde koştu. Fixture'ın
  "yazma yolu ön koşulları" baştan kurulmalıydı.

**İmprovize ettiğim, skill'in söylemesi gerekenler:**

- Fixture kurulumunu **ürünün kendi ORM'iyle** yazmak (ham SQL yerine): zorunlu
  alan/ilişki hatalarını okunur biçimde veriyor, iterasyonu hızlandırıyor.
- Salt-okunur paylaşılan fixture + mutasyon yapan case'e **kendi izole kopyasını
  yaratma** zorunluluğu ayrımını, ajan görev metnine açıkça yazmak.
- **Global durum gerektiren case'leri ana oturuma saklamak** (ör. "sistemde hiç
  X yok"): paralel ajanlardan hiçbiri koşamaz, çünkü ötekileri zehirler.

**Zorlanan non-negotiable:** "her FAIL'i kendin yeniden koş" kuralı, bu koşumda
en yüksek getirili kural oldu — üç ajan bulgusunun ikisi doğrulandı, biri
(yetki) çürütüldü, ve bir öncekinden kalan bir S1 iddiası ampirik olarak yanlış
çıktı. Kural pahalı ama vazgeçilmez.

**Öneriler:**

- **P1 — "Kontrat şemasını programatik tara" adımı, keşif charter'larının zorunlu
  bir alt maddesi olmalı.** `SKILL.md` Phase 1 kategori 15'e ve
  `references/backend-api.md`'ye: bir feature bir alanı/kaydı gizlemekle
  ilgiliyse, yayınlanan şemadan o alanı taşıyan tüm uçları çıkar ve her birini
  case'e dönüştür. Elle uç saymak bu sınıfı kaçırıyor.
- **P1 — Yayınlanan kontrat ile çalışma zamanı yanıtını karşılaştırma kuralı.**
  Aynı iki referans dosyaya: ilan edilen şema ile gerçek yanıt gövdesinin alan
  kümesi karşılaştırılmalı; fazla alan (ham entity dönmek) bir bulgudur — yeni
  kolon eklendiği gün sözleşmeye giriyor.
- **P2 — "Watch süreci ayaktayken derleme komutu koşma" uyarısı.**
  `references/backend-api.md` veya ortam manifesti şablonuna: derleme çıktısını
  paylaşan watch süreçleri, tam derleme koşulduğunda çöker; doğrulama turunda
  yalnızca test + tip kontrolü koşulur.
- **P2 — Fixture'ın yazma-yolu ön koşulları kontrol listesi.** `references/test-data.md`:
  fixture yalnızca okuma case'lerini değil, yazma case'lerinin ön koşullarını da
  (ücret bakiyesi, kota, izin kaydı) kurmalı; aksi halde ajanlar onları BLOCKED
  raporlar ve ana oturum aynı işi ikinci kez yapar.
- **P3 — Global durum gerektiren case'leri paralelleştirmeden muaf tutma.**
  `SKILL.md` paralelleştirme bölümüne: "sistemde hiç X yok" tipi case'ler ana
  oturuma saklanır, ajan grubuna verilmez.

**Açık öneri (onay bekliyor):** yukarıdaki P1/P1/P2/P2/P3. Bir önceki koşumun
önerileri de hâlâ onay bekliyor; **tekrar eden desen:** "harness kaynaklı sahte
FAIL"i azaltan kurallar iki koşumda da P1 çıktı — bu, onaylanmayı hak ettiğinin
göstergesi. Onaylanırsa MINOR: v1.7.0.

---

## 2026-08-27 · L2 · backend API (CQRS, kampanya/ödül alanı)

Öncesinde iki inceleme turu (biri sadeleştirme, biri statik kod incelemesi + düzeltme)
geçmiş bir değişiklik test edildi. Çıkan dört bulgunun **hiçbirini** o iki tur görmemişti;
ikisi diff'in tamamen dışından geldi.

**Skill'in yapısı ne yakaladı:**
- **Kategori 13 (mass assignment probu) tek başına bir S1 buldu:** yetkilendirme URL
  parametresine göre yapılırken gövdedeki aynı adlı alanın kazanması, yetki kontrolünü
  baypas eden bir yazma üretiyordu. Feature'ın kendisiyle hiç ilgisi yoktu — katalog satırı
  zorlamasa test edilmeyecekti.
- **Kategori 9'un "cross-user paralel" ayrımı** bir S2 buldu: paylaşılan bütçe cap'i aynı
  kullanıcıda tutuyor, farklı kullanıcılarda tutmuyordu. "Eşzamanlılık" başlığını "aynı
  kayıt" diye okuyup geçmek bu sınıfı kaçırıyor.
- **Kategori 19 + ortamın geri kalmışlığı:** hedef ortam migration'ı henüz uygulamamıştı;
  bu, veri dönüşüm yolunu gerçek veriyle test etmek için tek pencereydi.

**Ne zaman/gürültü üretti:**
- Ajanlara verdiğim ortam ipuçlarını (listeleme ucunun koordinat gerektirmesi) kendi
  ölçümümde kullanmadım → bir sahte FAIL. Ajan talimatı ana oturumdan daha bilgiliydi.
- Tam derleme komutu, ayakta duran watch sürecini öldürdü (bir tur kayıp) — bu zaten
  önceki koşumun P2 önerisiydi, onaylanmadığı için tekrarladı.

**Doğaçlama ettiklerim (kural olması gerekenler):**
1. **Bir güvenlik bulgusunun etkisi ilk engelde ölçülmez.** Yetkisiz hedefe yazma denemesi
   ikincil bir kontrolden (kaynak-sahiplik kontrolü) 400 aldı; orada durmak "etki sınırlı"
   derdi. O kontrolü baypas eden ikinci bir yol (her tenant'a açık paylaşılan kaynak)
   denendiğinde bulgu S2'den **S1'e** çıktı. Kural: bir baypas bulgusunda, ilk reddi üreten
   ikincil kontrolün kendisi de baypas edilmeye çalışılmalı; severity ancak ondan sonra yazılır.
2. **Veri dönüştüren bir migration, ancak uygulanmadan önce test edilebilir.** Dönüşecek
   veriyi migration'dan ÖNCE yaratmak gerekir; uygulanmış bir ortamda o yol artık
   gözlemlenemez ve "migration çalıştı" ifadesi dönüşümü değil yalnızca şema değişimini kanıtlar.
3. **Ajan talimatına yazılan ortam ipucu, ana oturumun kendi koşumu için de geçerlidir.**
   Aynı ipucu iki yere yazılmıyorsa ana oturum kendi sahte FAIL'ini üretiyor.
4. **Bir düzeltmenin kalıcı testi yoksa düzeltme yarımdır.** Controller seviyesindeki bir
   sıra düzeltmesinin regresyon testi projede yoktu; testi yazıp, düzeltmeyi geçici geri
   çevirerek kırmızı olduğunu kanıtlamak (yalnız "yeşil" görmek değil) testin gerçekten
   assert ettiğini gösterdi.

**Non-negotiable gözlemi:** #3 (red-green) bir controller düzeltmesinde zorlandı çünkü
projede o katmanın test kalıbı yoktu. Kuralı esnetmek yerine kalıbı kurmak doğru çıktı —
ama skill, "düzeltme katmanının test kalıbı yoksa onu kurmak da düzeltmenin parçasıdır"
demiyor.

**Öneriler:**
- **P1 — Baypas bulgularında ikincil kontrolü de test etme kuralı** (yukarıdaki 1).
  `SKILL.md` Phase 3'e: severity, ilk engelin ardında durmadan, o engeli baypas eden ikinci
  yol denenerek yazılır.
- **P1 — Veri dönüştüren migration'ın test penceresi** (yukarıdaki 2). `SKILL.md` kategori 19
  ve `references/backend-api.md`: dönüşüm case'i migration uygulanmadan önce kurulur; ortam
  zaten uygulanmışsa bu case `BLOCKED` yazılır, `PASS` değil.
- **P2 — Ortam ipuçlarının tek kaynağı** (yukarıdaki 3). Ajan talimatına giren her ortam
  ipucu aynı anda ortam manifestine yazılır; ana oturum kendi case'lerini o manifestten okur.
- **P2 — Düzeltmenin test kalıbı yoksa kalıbı kurmak düzeltmenin parçasıdır** (yukarıdaki 4).
  `SKILL.md` Phase 5'e.

**Açık öneri (onay bekliyor):** yukarıdaki P1/P1/P2/P2 + önceki iki koşumun onay bekleyen
önerileri. **Üçüncü kez tekrar eden desen:** "watch süreci ayaktayken tam derleme koşma"
uyarısı bu koşumda fiilen zaman kaybettirdi — üç koşumdur P2 olarak duruyor, artık P1
sayılmalı. Onaylanırsa MINOR: v1.7.0.

---

## 2026-09-01 — L3, backend API (ikinci giriş noktası / tool yüzeyi)

- **Case:** 231 · **Sonuç:** 214 PASS / 10 FAIL (6× S2, 7× S3, 5× S4) / 3 açık soru /
  7 NOT RUN-BLOCKED · **Karar:** NO-GO · **Skill sürümü:** 1.6.0 · **Yürütme:** 7 paralel
  Sonnet grubu + lider doğrulaması

**Genelleştirilmiş dersler:**

- **Katalog satırı 15 (hata tahmini / keşif) koşunun en ciddi üç bulgusunu tek başına üretti**
  (6 case → 3 FAIL, hepsi S2). Matrisin üretebildiği kategoriler yüzeyin sözleşmesini doğruladı;
  sözleşmenin *kendi içinde tutarsız* olduğu yerleri yalnızca timeboxed keşif buldu. Satır 15'in
  "always" işaretini hak ettiğinin en net kanıtı.
- **Bir okuma yolunda güvenli olan bir sınır/guard, yazma yolunda ters etki yapar.** Yanıtı
  reddetmek okumada "eksik cevap verme" demektir; yazmada "yan etki gerçekleşti ama olmadı denildi"
  üretir — çağıran tekrar dener ve yinelenen kayıt oluşur. Koşunun en ciddi bulgusu buydu ve hiçbir
  kategori satırı bunu doğrudan sormuyor.
- **Bir feature mevcut mantığa ikinci bir giriş noktası eklediğinde, iki giriş noktasının BEYAN
  ETTİĞİ korumaları kaynak seviyesinde diff'lemek gerekiyor.** Siyah kutu testi "reddedildi mi"
  sorusunu cevaplar, "doğru koruma mı seçildi" sorusunu cevaplamaz. Bu diff bir yetki yükseltme
  bulgusu (S2) üretti: bir gövde alanı, kendisi için ayrı bir izin tanımlanmış bir lifecycle
  geçişini daha zayıf bir izinle tetikliyordu.
- **Aynı veri sınıfı için iki maskeleme/koruma yüzeyi varsa, biri eksiktir.** İki yüzeyin
  uyuşmazlığı bulgunun kendisi olur ve tartışmayı bitirir: "kasıtlı mı" sorusuna, projenin aynı
  değeri başka yerde maskelemesi cevap verir.
- **Aynı sınıftaki iki girdi farklı zamanlarda doğrulanıyorsa, geç doğrulanan sessizce üretime
  kaçar.** Bir tür referans create anında reddedilirken kardeşi yalnızca çalışma anında
  başarısız oluyordu — ve dry-run doğrulama aracı ikincisine temiz rapor veriyordu.
- **Paralel yürütmede fixture izolasyonu VERİYLE SINIRLI KALMAMALI: kimlik ve yetki nesneleri de
  izole olmalı.** İki grup aynı yetki grubunu mutasyona uğratacak şekilde görevlendirilmişti; koşu
  ortasında düzeltme mesajı göndermek gerekti. Sahte FAIL riski en pahalı olduğu kategoride
  (güvenlik/yetki) doğuyor.
- **Yürütücülere verilen brief'te adı geçen her ortak artefakt, dispatch'ten ÖNCE gerçekten
  yazıyor mu diye kontrol edilmeli.** Brief'te "her çağrı şu dosyaya birikir" denen dosyaya
  harness hiç yazmıyordu; bir ajan profili elle yeniden kurmak için ciddi efor harcadı.
- **Ortak, case-id başına otomatik kanıt arşivleyen bir harness'i dispatch'ten önce kurmak,
  "her case için ham kanıt" kuralını disiplin meselesinden bedava bir yan etkiye çevirdi.** 7 grup
  boyunca tek bir "kanıt yok" durumu çıkmadı.
- **Kendi vaka listem 3 sahte FAIL üretti:** yanlış bir sabit değer beklentisi, var olmayan bir
  şema kolonunun varsayılması, ve kurulum adımının hatasını yutan bir script. Üçü de Phase 3'te
  öldü — ama üçü de vaka tasarımı sırasında, tek bir "beklentiyi nereden aldım" sorusuyla
  önlenebilirdi.
- **Eksik bir özelliği hata olarak raporlama baskısı gerçek:** hiçbir yerde vaat edilmemiş bir
  tekillik kuralının yokluğu, bir yürütücü tarafından "dokümanla çelişiyor" diye raporlandı.
  "Dayanak yoksa bulgu değil, açık sorudur" kuralı bunu tuttu.

**Öneriler:**

- **P1 — Fixture izolasyon kuralını kimlik/yetki nesnelerine genişlet.** SKILL.md'nin paralel
  yürütme bölümündeki "her ajan kendi izole fixture setini seed eder" maddesi yalnız veriden
  bahsediyor. Kimlik, rol, izin grubu, API anahtarı gibi *yetkilendirme* nesneleri de aynı kurala
  girmeli; aksi halde iki grup birbirinin izin durumunu değiştirir ve sahte FAIL tam olarak
  güvenlik kategorisinde çıkar. (Bu koşuda koşu ortasında düzeltme gerekti.)
- **P1 — `references/oracles.md`'ye yeni bir sezgi: "ikinci giriş noktası diff'i".** Bir feature
  mevcut mantığa ikinci bir giriş noktası (tool yüzeyi, kuyruk tüketicisi, batch iş, admin CLI)
  eklediğinde, o giriş noktasının beyan ettiği korumaları birincinin beyanlarıyla kaynak seviyesinde
  karşılaştır. Siyah kutu "reddedildi mi"yi görür, "doğru koruma mı"yı görmez. Bu koşuda S2 üretti.
- **P2 — `references/oracles.md`'ye: "okuma yolunda doğrulanmış bir sınır, yazma yolunda yeniden
  test edilmeli".** Guard/limit/kesme davranışları yan etkinin öncesinde mi sonrasında mı çalıştığı
  sorusuyla birlikte test edilmeli; yan etkiden sonra reddetmek "sessizce tamamlandı ama başarısız
  raporlandı" üretir. Bu koşunun en ciddi bulgusu.
- **P2 — Phase 2'ye bir satır: paralel dispatch'ten önce ortak harness'i ve brief'te adı geçen her
  ortak artefaktı bir çağrıyla doğrula.** İsmi geçen ama yazmayan bir dosya, ajan başına ölçülebilir
  efor kaybı demek.
- **P3 — Rapor şablonuna "paylaşılan durumu mutasyona uğrattığı için koşulmayan süitler" için
  adlandırılmış bir bölüm.** Bu koşuda §6 olarak doğaçlandı; `BLOCKED (ortam)` ile aynı şey değil
  (ortam eksik değil, koşmak *başka* şeyi bozacaktı) ve devredilen borç listesine farklı bir
  gerekçeyle giriyor.

### Aynı koşunun düzeltme fazı (aynı gün, 18 düzeltme)

- **Bir düzeltmeyi yalnız unit testle doğrulamak, düzeltmenin hiç çalışmadığını gizleyebilir.** Bir
  düzeltme, aynı verinin iki farklı CLR şeklinde geldiği bir yolda no-op çıktı (bellekte kurulan
  değer vs. serileştirmeden dönen değer); düzeltilen iki yüzeyde hiç çalışmıyordu ama üçüncü yüzeyde
  çalıştığı için tüm unit testler yeşildi. **Sadece bulguyu üreten canlı probun tekrar koşulması
  yakaladı.** Kural: her düzeltme, bulguyu üreten prob ile — testle değil — kapatılmalı.
- **Düzeltme fazının kendi doğrulama koşusu da yanlış alarm üretir, ve oranı yüksektir.** Bu fazda 4
  "FAIL"ın 3'ü benim assertion hatamdı (unicode-escape'li JSON'da düz string arama, `psql`'in
  boolean'ı `f` değil `false` yazması, eşiğe ulaşmayan test verisi). Ham çıktıyı yazdırmak üçünü de
  dakikalar içinde ayırdı; paraphrase etseydim üçü de bulgu olarak raporlanabilirdi.
- **Mevcut testlerin düzeltme sonrası kırılması en temiz kırmızı-yeşil kanıtıdır.** Üç düzeltmede
  eski davranışı pinleyen testler kırıldı (biri test ADINDA eski beklentiyi taşıyordu). Bunları
  "yeni sözleşmeye güncellemek" assertion zayıflatmak değildir — ama ayrımı raporda açıkça yazmak
  gerekiyor, yoksa okuyucu ikisini ayırt edemez.
- **Bir subagent, brief'te yasaklanmadığı için yıkıcı bir git komutu çalıştırdı** (`git checkout --
  <file>`, staged olmayan değişiklikleri atan). Bu koşuda veri kaybı olmadı — koşu başındaki durum ve
  o anki diff birlikte bunu kanıtladı — ama kanıtlamak zorunda kalmak, brief'in eksik olduğunun
  kanıtı.

**Ek öneriler:**

- **P1 — Yürütücü/uygulayıcı subagent brief'ine yıkıcı komut yasağı eklenmeli.** `git checkout --`,
  `git reset`, `git stash`, `git clean`, dosya silme: hiçbiri onay alınmadan çalıştırılmamalı, ve
  "kendi geçici değişikliğimi geri alıyorum" bir istisna değildir (paylaşılan bir dosyada başkasının
  kaydedilmemiş değişikliğini de alır). SKILL.md'nin subagent bölümünde tek satır.
- **P1 — "Düzeltmeyi bulgunun kendi probuyla kapat" kuralı Phase 5'e girmeli.** Şu an Phase 5
  "regresyon testi ekle + yeniden koş" diyor; testin *doğru şeyi* koştuğunun garantisi yok. Bulguyu
  üreten prob, testin yerine değil, testin yanında koşulmalı.
- **P2 — Rapor şablonu, "mevcut bir testin düzeltme sonrası kırılıp yeni sözleşmeye güncellenmesi"
  ile "yeni yazılmış test" arasını ayırmalı.** İlki kırmızı-yeşil kanıtıdır ve en değerli satırdır;
  aynı listede görünmeleri o kanıtı görünmez yapıyor.


---

## 2026-09-07 · L3 · tarayıcı içi LLM istemcisi + web frontend (düzeltme modu açık)

**Yakalananlar (yapı sayesinde):**

- **Kategori katalogunun "yerelleştirme/uç veri" satırı,** özellik listesinde hiç geçmeyen bir
  belge-dili hatası buldu: belge dili UI dilini takip etmediği için CSS `uppercase` yanlış dilin
  büyük harf kurallarını uyguluyordu ve dilin en sık harflerinden biri bozuk basılıyordu. Hiçbir
  fonksiyonel senaryo buna bakmaz; satır olduğu için bakıldı.
- **"Basis'i kaynak koddaki niyet yorumlarından da oku" yaklaşımı** iki S2'yi doğrudan üretti. Bu
  kod tabanında niyet yorumlarda açıkça yazılı olduğu için karşılaştırma yapılabildi: yorumun
  vaat ettiği ile kodun yaptığı arasındaki fark, dokümansız bir projede görünmez olurdu.
- **"Beklenmeyen FAIL kümesinde önce kendi harness'ından şüphe et"** kuralı üç kez ateşlendi ve
  üçünde de haklıydı (maskeleme regex'i kendi kontrol alanını gizledi; yerinde mutate edilen bir
  dizi assertion'ı bozdu; paylaşılan bir test yardımcısı prop'u yutuyordu). Üçü de rapora hiç
  girmedi.

**Maliyet/gürültü:**

- **Seviye tablosu, "yürütmesi para yakan sistem" durumunu hiç hesaba katmıyor.** Burada her
  senaryo gerçek bir ücretli model çağrısıydı. Case listesi yürütme maliyetine göre sınıflandırıldı
  (deterministik / arayüz / ücretli) ve derinlik ucuz olan yere kaydırıldı — bu doğaçlamaydı, kural
  değil.
- **Aynı koşuda "hepsini düzelt" modu, kapsamı sessizce takas etti.** 12 bulgunun 11'i kapandı ama
  tasarlanan 122 case'in 57'si hiç koşulmadı; eşzamanlılık, dayanıklılık ve keşif charter'larının
  **tamamı** koşulmayanlar arasında. Seviye tablosu L3 için "60-120 case koşulur" diyor; bu koşu o
  vaadi tuttuğu izlenimi veriyor ama tutmuyor.

**Doğaçlananlar (kural olmalı):**

- Bağımlılığın **ilan ettiği** yetenekler ile istemcinin **tükettiği** yetenekler arasında kaynak
  seviyesinde fark alma. Bu koşunun en ciddi bulgusu buradan çıktı ve siyah kutuya tamamen
  görünmezdi: hiçbir şey hata vermiyordu, sistem yalnızca sessizce kötü çalışıyordu.
- Değiştirilen bir varsayılan/fabrika fonksiyonu için, **o fabrikayı mock'layan testleri** ara.
  Tam olarak onlar yeşil kalacak testlerdir.

**Zorlanan non-negotiable:**

- **#4 (mevcut davranışı değil niyeti test et)** iki yerde zorlandı ve ikisi farklı sebepten:
  (a) mevcut bir test hatalı davranışı bilinçli olarak pinlemişti (yorumu bunu açıkça yazıyordu);
  (b) yazılı basis'in kendisi, sonradan gelen bir kullanıcı kararıyla geçersiz kılınmıştı ve dosya
  bunu bilmiyordu. İkincisi kuralda hiç ele alınmamış bir durum.

**Öneriler:**

- **P1 — "Basis bayat olabilir" kuralı, `references/oracles.md` ve Phase 0'a.** Yazılı bir basis
  (plan, spec, tasarım kararı) projedeki **sonraki** bir talimatla geçersiz kılınmış olabilir. Basis
  bir artefakt ise, onu geçersiz kılan daha yeni bir karar var mı diye kontrol et; varsa en yeni
  karar geçerlidir ve bulgu **koda değil dokümana** yazılır ("plan bu noktada bayat"). Aksi halde
  doğru kod, güvenle FAIL raporlanır. SKILL.md Phase 0 zaten "basis'in kendisi kusurlu olabilir"
  diyor ama yalnız *çelişki/eksiklik* için; *sonradan geçersiz kılınma* farklı bir durum ve daha
  sinsi.
- **P1 — `references/oracles.md`'ye yeni sezgi: "ilan edilen yetenek / tüketilen yetenek diff'i".**
  Sistem, yeteneklerini ilan eden bir bağımlılıkla konuşuyorsa (protokol yetenekleri, şema
  ipuçları, annotation'lar, olay tipleri, sayfalama meta'sı, cache başlıkları), ilan edilenlerin
  hangilerinin kodda **tüketildiğini** kaynak seviyesinde karşılaştır. Tüketilmeyen bir ilan hata
  vermez; sistem yalnızca sessizce kötü çalışır — ve bağımlılığın dokümanı istemciyi o yola
  yönlendiriyorsa, tüketici o talimatı izleyemez.
- **P2 — Seviye tablosuna yürütme maliyeti boyutu.** Kickoff'ta, yürütmesi para yakan (ücretli API,
  gerçek dış çağrı, uzun süren iş) senaryolar için case'leri maliyet sınıfına ayır ve derinliği
  ucuz sınıfa kaydır. Rapor, koşulan case'lerin maliyet dağılımını da yazsın — yoksa "L3 koştum"
  ifadesi maliyeti çok farklı iki koşu için aynı şeyi ifade ediyor.
- **P2 — "L3 + düzeltme modu" için açık bir uyarı.** İkisi aynı koşuda seçildiğinde, kickoff'ta
  kapsamın takas edileceği **söylenmeli** ("keşif mi kapanış mı önce?") ya da koşu ikiye
  bölünmeli. Şu anki metin bu kombinasyonun kapsam üzerindeki etkisini hiç anmıyor.
- **P3 — Değişen bir varsayılan/fabrika için "o fabrikayı mock'layan testleri ara" satırı,**
  Phase 0'ın mevcut testleri okuma maddesine. Bu koşuda gerçek bir kör nokta buldu: 35 test
  fabrikayı mock'layıp değişen alanı enjekte ediyordu, dolayısıyla değişikliğin kırdığı hiçbir şey
  görünmeyecekti.


---

## 2026-09-16 · L3 · backend API, çok kiracılı yetki sınırı (rapor-only; koşumu yapan = değişikliği yazan)

**Case:** 138 · **Sonuç:** 122 PASS / 3 FAIL / 13 BLOCKED · **Karar:** GO WITH RISK
(değişikliğe atfedilebilen 0 bulgu; açık 2×S2 önceden var) · **Skill sürümü:** 1.7.0

**Yakalananlar (yapı sayesinde):**

- **Kategori 6 (kombinasyonlar)** karar tablosu olarak kurulunca "yüzey × enjeksiyon
  noktası × kiracı ilişkisi" matrisi çıktı ve taramayı 2–3 uçtan 14 yüzeye taşıdı.
  Doğaçlama bir koşu bunların ilk üçünde durur.
- **Non-negotiable #8 (kiracı izolasyonu bir kategori değil, ayakta duran bir iddia)**
  iddianın şeklini değiştirdi: "403 mü döndü" yerine "kayıt hangi kiracıya düştü".
  Bu yeniden çerçeveleme, koşumun en ağır bulgusunu (ödeme, başka kiracının alt
  kaydına bağlanabiliyor) doğrudan üretti — ve bu arada 403 bekleyen bir iddianın
  düzeltme geri alınsa bile geçmeye devam edeceğini de gösterdi.
- **Kategori 4 + tutarlılık oracle'ı** kardeş uçların aynı sınır değerinde farklı
  davrandığını yakaladı (biri 400, diğeri 500). Keşifle bulunmaz.
- **Basis'i tersten okuma** kuralı, silinmiş bir alanın hiçbir case tarafından
  kapsanmadığını gösterdi; açık koşum içinde kapatıldı.

**Maliyet/gürültü:**

- Bir yürütücü ajan, gerçek bir dış sağlayıcı çağrısı yapmaktan çekinip case'i
  NOT RUN bıraktı; sağlayıcı aslında erişilebilir ve güvenliydi, case iki çağrıda
  tamamlandı. Ortam manifesti bağımlılığın *var olup olmadığını* yazıyor,
  *çağrılmasının güvenli olup olmadığını* yazmıyor.
- İki sahte FAIL, ikisi de aynı sebepten: enum'ın tanımlayıcı adı wire değeri
  sanılarak kullanıldı. Mevcut "sentetik değer geçerli olmalı" kuralı format
  kontrolünden bahsediyor; bu hata format kontrolünü geçiyor çünkü şekil doğru.
- 138 case'in 13'ü BLOCKED ve neredeyse tamamı **veri** eksikliği. Ortam envanteri
  bağımlılıkları kapsıyor, fixture'ları kapsamıyordu; hepsi koşum ortasında
  keşfedildi.

**Doğaçlananlar (kural olmalı):**

- **A/B diferansiyel koşum.** Değişiklik öncesi build'i aynı veri deposuna karşı
  yan yana ayağa kaldırmak ve her bulguyu iki tarafta da koşmak. Koşumun tek en
  değerli hamlesiydi: "sekiz bulgunun sekizi de önceden var" cümlesi iddia değil
  ölçüm oldu, dokunulmaması gereken yüzeyler bayt bayt karşılaştırılabildi ve
  düzeltmenin kırmızı-yeşili ayrı bir test yazmadan çıktı.
- **Temelin geçerliliğini önce kanıtlama.** Eski sürecin başlangıç zamanının ilk
  edit'ten önce olduğu gösterilmeden hiçbir A/B sonucuna güvenilmedi. Bu adım
  olmadan teknik kendi sonucunun tersini kanıtlayabilir.

**Zorlanan non-negotiable:**

- **#6 (kendini sertifikalandırma yasağı)**, kuralın adını koymadığı bir yerden
  zorlandı: değişikliği yazan kişi case listesini de tasarladı. Kural yürütme
  bağımsızlığını istiyor, tasarım bağımsızlığını istemiyor. Kendi yazdığı yeni
  kodun sınır değer tutarsızlığını, listeyi tasarlayan değil, aynı uçları kardeş
  bir uçla karşılaştıran bağımsız bir ajan buldu.
- **Karar kuralı gerçeklikle çarpıştı:** "kritik akışta açık S2 → NO-GO", önceden
  var olan bir S2 yüzünden, kanıtlanmış bir S1'i kapatan düzeltmeyi bloke
  ediyordu. Kural kimin bulgusu olduğunu sormuyor.

**Öneriler:**

- P1 — diferansiyel (A/B) yürütme tekniği + temel geçerlilik kanıtı → UYGULANDI (v1.8.0)
- P1 — karar kurallarına atıf boyutu → UYGULANDI (v1.8.0)
- P2 — Phase 0 fixture envanteri → UYGULANDI (v1.8.0)
- P2 — non-negotiable #6'ya tasarım bağımsızlığı → UYGULANDI (v1.8.0)
- P3 — sentetik değer kuralına "değeri oku, ismi değil" → UYGULANDI (v1.8.0)
- **P3 — AÇIK:** ortam manifestinin bağımlılık tablosuna "çağrılması güvenli mi"
  sütunu. Şu an yalnızca *gerçek / mock / yok* yazıyor; yürütücü ajan bir dış
  çağrının geri döndürülemez maliyeti olup olmadığını bilmediği için temkinli
  davranıp case'i koşmuyor. Sonraki koşumda tekrarlarsa P2'ye yükselir.
