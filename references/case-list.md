# The case list: format, IDs, statuses, lifecycle

The case list is the artefact that makes a run auditable instead of a story about
a run. Write it before execution, keep it updated during execution, and hand it
over with the report.

## Where it lives

- With QA memory: `.qa/suites/<feature>.md` — reused and extended on later runs,
  so it grows into the project's case inventory.
- Without `.qa/`: `test-cases-<feature>-<YYYY-MM-DD>.md` next to the report.

One file per feature/area, not per run. Runs are recorded inside it (see
`## Koşum geçmişi` below) and in `.qa/regression-log.md`.

## Format

```markdown
# Test Case Listesi — Kupon indirimi
Branch/commit: feature/coupon @ abc1234 | Ortam: local | Yazan: qa-engineer
Tier planı: A=kupon indirimi · B=sipariş toplamı, stok · C=login+sipariş smoke · D=webhook

## Özet
Toplam 47 case — 5 happy, 8 fonksiyonel, 11 negatif, 9 sınır, 4 yetki,
6 durum/eşzamanlılık, 4 veri bütünlüğü. Atlanan: #17 a11y (UI değişmedi),
#16 performans (dev DB'de tek kayıt var).

## Case'ler

| ID | Tier | Kategori | Senaryo | Ön koşul / veri | Adımlar | Beklenen | Durum | Kanıt |
|----|------|----------|---------|-----------------|---------|----------|-------|-------|
| TC-001 | A | Happy | Geçerli kupon %10 indirim uygular | aktif kupon `SAVE10`, sepet 100₺ | POST /orders (coupon=SAVE10) | 201, total=90₺, DB'de discount=10 | PASS | resp 201, order#881 |
| TC-014 | A | Sınır | Kupon tutarı sepet toplamına eşit | kupon 100₺, sepet 100₺ | POST /orders | 201, total=0, negatife düşmez | FAIL (S2) | total=-0.01 → BUG-016 |
| TC-021 | A | Eşzamanlılık | Aynı kupon 20 paralel istekte | tek kullanımlık kupon | 20× POST paralel | 1 başarılı, 19 reddedilir | FAIL (S1) | 3 başarılı → BUG-017 |
| TC-033 | B | Veri bütünlüğü | İndirim iptali sonrası toplam tutarlı | TC-001'in siparişi | DELETE /orders/881 | stok geri, discount kaydı silinir | PASS | DB kontrol edildi |
| TC-041 | C | Kritik akış | Login smoke | — | POST /auth/login | 200 + token | PASS | |
| TC-048* | A | Keşifsel | Kupon süresi istek sırasında doluyor | kupon 5sn sonra biter | POST /orders (t=6sn) | 400 expired | PASS | koşum sırasında eklendi |

## Koşum geçmişi
| Tarih | Commit | Koşulan | PASS | FAIL | BLOCKED | NOT RUN | Karar |
|-------|--------|---------|------|------|---------|---------|-------|
| 2026-08-20 | abc1234 | 47 | 43 | 3 | 1 | 0 | NO-GO |
```

## ID rules

- `TC-001` upward, zero-padded, **never renumbered**. A finding, a regression test
  and next month's re-run all point at the same ID.
- New cases on a later run continue the sequence — don't reset, don't reuse the ID
  of a deleted case.
- Suffix `*` marks a case discovered *during* execution rather than designed up
  front. Worth keeping visible: a high `*` count means Phase 1 was too shallow,
  and that's useful feedback about your own design.
- Reference IDs everywhere: findings (`TC-021 → BUG-017`), regression tests
  (`test_concurrent_redeem  # TC-021`), the report's coverage table.

## Status values

| Status | Meaning |
|--------|---------|
| *(empty)* | Designed, not yet run |
| `PASS` | Executed; observed result matched the expectation |
| `FAIL (Sn)` | Executed; mismatch, verified in Phase 3, linked to a finding |
| `BLOCKED` | Could not run for an environmental/access reason — state it |
| `NOT RUN` | Deliberately skipped this run — state why (out of scope, deprioritised) |
| `SKIP-N/A` | Doesn't apply to this codebase (no UI, no auth layer, …) |

Never leave a row without a status, and never write `PASS` for something you
didn't watch happen. `BLOCKED` and `NOT RUN` are honest; a fabricated `PASS` is
the one thing that makes the whole file worthless.

## Writing good cases

- **One assertion per case.** "Geçersiz e-posta reddedilir *ve* hata mesajı
  Türkçe" is two cases; when it fails you want to know which half broke.
- **Concrete data, not descriptions.** `SAVE10`, `100₺`, `1000 karakter` — not
  "geçersiz değer". Someone else (or you, next month) must be able to re-run it
  without re-deriving the inputs.
- **Expected must be checkable and anchored.** Not "hata verir" but "400 +
  `code=COUPON_EXPIRED`, sipariş oluşmaz". If nothing in the requirements or
  schema anchors your expectation, that's an open question, not a case.
- **Include the verification point**, not just the request: which DB row, which
  log line, which counter you'll look at.
- **Steps short enough to repeat.** If a case needs 12 steps of setup, the
  precondition column is doing the wrong job — split it.

## Lifecycle across runs

1. **First run:** design the list, deliver it, execute it, append the run to
   `## Koşum geçmişi`.
2. **Later runs on the same area:** re-run the existing cases as the regression
   suite, then append new cases for whatever the new diff introduced. Mark cases
   that no longer apply as `SKIP-N/A` with a note — delete rather than leave a
   silently stale expectation only when the feature is genuinely gone.
3. **When a case keeps catching things**, promote it to an automated test and note
   the test's path in the row. The case list is the backlog of tests worth
   automating; the ones that never fail are cheap to keep as manual checks.
