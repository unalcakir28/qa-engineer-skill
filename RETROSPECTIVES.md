# qa-engineer — Koşum retrospektifleri

Her koşumun sonunda (Phase 7) bir girdi. Öneriler P1 (yanlış sonuca yol açtı) /
P2 (ciddi zaman kaybı) / P3 (cila) olarak derecelendirilir. Tekrarlayan öneri
otomatik P1'e yükselir. Uygulanan öneriler `CHANGELOG.md`'ye taşınır ve burada
`UYGULANDI (vX.Y.Z)` olarak kapatılır.

---

## 2026-08-20 — tgn-network / TGNL-247 kampanya ödül ret defteri

- **Seviye:** L3 (kullanıcı seçti) · **Case:** 120 · **Sonuç:** 117 PASS / 1 FAIL
  (S2, düzeltildi) / 1 BLOCKED / 1 SKIP · **Karar:** GO · **Skill sürümü:** 1.0.0

**1. Yapı ne yakaladı?** S2'yi (kapanmış ret kaydının yeni retle yeniden
açılmaması) kataloğun 8. satırı (durum & sıra) yakaladı — doğaçlama testte
yazılmayacak bir case'ti ve feature'ın ana vaadini (`resolved=false` = "şu an
fiilen başarısızlar") sessizce bozuyordu. Kırmızı-yeşil kuralı (non-negotiable
#3) düzeltme testinin gerçekten bir şey kanıtladığını garanti etti; "kendini
sertifikalandırma" yasağı (#6) düzeltmenin gerçek HTTP ile ikinci kez
doğrulanmasını zorladı.

**2. Süreç nerede zaman kaybettirdi?**
- Senaryolar birbirinin verisini kalıcı bozdu (pasif kampanya aktifleştirildi,
  süresi dolan uzatıldı) → 3 sahte FAIL, bir tur yeniden koşum.
- Sentetik truId `@IsUUID()`'den döndü → tüm auth çağrıları sahte 400; neredeyse
  kod bug'ı olarak raporlanıyordu.
- TC-099 FAIL'i subagent'ın kendi hatasıydı (zorunlu `conversationId` eksik).
- TruID gRPC'nin lokalde olmadığı koşum ortasında keşfedildi → bulk case'lerine
  boşa subagent tokenı.

**3. Ne doğaçlandı?** Ajan başına izole fixture seti; response-body loglama +
validasyon/business 400 ayrımı; BLOCKED case'lerin "preprod'da koşulacak" notuna
çevrilmesi; suite için core/rotasyon ayrımı fikri.

**4. Zorlanan kural?** Yok — ama reporting.md'deki severity rubrik'inin
SKILL.md'den görünür olmaması, S2/S3 kararının içgüdüyle verilmesine yol açtı
(rubrik sonradan bulundu; kural değil, keşfedilebilirlik sorunu — P3).

**Öneriler:**
- P1 — fixture izolasyonu kuralı → UYGULANDI (v1.1.0)
- P1 — harness-first şüphe refleksi → UYGULANDI (v1.1.0)
- P2 — subagent ham kanıt + PASS örneklemesi → UYGULANDI (v1.1.0)
- P2 — Phase 0 ortam yetenek envanteri → UYGULANDI (v1.1.0)
- P2 — seviye seçeneklerine maliyet etiketi → UYGULANDI (v1.1.0)
- P3 — BLOCKED borç takibi → UYGULANDI (v1.1.0)
- P3 — suite budama stratejisi → UYGULANDI (v1.1.0)
- P3 — severity rubrik'i önerisi KAPANDI: rubrik zaten `references/reporting.md`'de
  varmış; sorun keşfedilebilirlikti, Phase 4 zaten oraya yönlendiriyor.

**Açık öneri (onay bekliyor):** yok.
