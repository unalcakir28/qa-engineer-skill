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
