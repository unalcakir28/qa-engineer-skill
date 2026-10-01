# qa-engineer — Claude Code skill

Project-agnostic QA engineer skill: risk analysis → technique-based case list
(20-category catalogue) → run (parallel Sonnet subagents) → finding
verification → severity-ranked report + GO / NO-GO verdict. Every run
evaluates itself at the end (Phase 7) and is versioned with approved
proposals.

## Install

```bash
git clone git@github.com:unalcakir28/qa-engineer-skill.git ~/.claude/skills/qa-engineer
```

In Claude Code, `/qa-engineer` (or just saying "test this" is enough — the
skill triggers itself).

## Structure

- `SKILL.md` — the whole process (kickoff, Phase 0–7, category catalogue,
  rules)
- `references/` — techniques, oracles, checklists, templates, test-data
  guide
- `CHANGELOG.md` — semver release history; each entry says which run's
  retrospective it came from
- `RETROSPECTIVES.md` — per-run self-evaluation log

Project-specific information does not go into this repo — it lives in each
project's own `.qa/` folder (see `references/qa-memory.md`).
