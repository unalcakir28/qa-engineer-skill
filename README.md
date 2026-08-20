# qa-engineer — Claude Code skill

Proje-bağımsız QA mühendisi skill'i: risk analizi → teknik-tabanlı case listesi
(20 kategorilik katalog) → koşum (paralel Sonnet subagent'ları) → bulgu
doğrulama → severity'li rapor + GO / NO-GO kararı. Her koşum sonunda kendini
değerlendirir (Phase 7) ve onaylı önerilerle sürümlenir.

## Kurulum

```bash
git clone git@github.com:unalcakir28/qa-engineer-skill.git ~/.claude/skills/qa-engineer
```

Claude Code'da `/qa-engineer` (veya "test et" demek yeterli — skill kendini
tetikler).

## Yapı

- `SKILL.md` — sürecin tamamı (kickoff, Phase 0–7, kategori kataloğu, kurallar)
- `references/` — teknikler, oracle'lar, checklistler, şablonlar, test-data
  rehberi
- `CHANGELOG.md` — semver sürüm geçmişi; her girdi hangi koşumun
  retrospektifinden geldiğini söyler
- `RETROSPECTIVES.md` — koşum başına öz-değerlendirme günlüğü

Proje-spesifik bilgi bu repoya girmez — o, her projenin kendi `.qa/`
klasöründe yaşar (bkz. `references/qa-memory.md`).
