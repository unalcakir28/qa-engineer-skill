# Maintaining the skill — retrospective entries, generalisation, versioning

Read this in Phase 7 once the four retrospective questions have been answered and
there is something worth writing down. The four questions themselves stay in
`SKILL.md`; this file is what happens to the answers.

## The retrospective entry

Append one entry per run to `RETROSPECTIVES.md` in this skill's directory — but
keep it a **proposals ledger, not a run diary**. The run's story (what was
tested, which bugs, which project) already lives in that project's `.qa/`;
duplicating it here would smuggle project data into a project-agnostic repo.

An entry identifies the run only by **date + level + surface type**
("2026-08-20, L3, backend API") and contains:

- one generalised line per lesson, and
- the improvement proposals, each with a severity of its own:
  - **P1** — the skill caused a wrong result or a real risk
  - **P2** — significant wasted effort
  - **P3** — polish

If a lesson can't be generalised, it isn't a skill lesson — it goes to the
project's `.qa/` instead.

**Check the backlog first.** Before proposing, re-read the open proposals in
`RETROSPECTIVES.md`. A proposal that recurs across runs is prima facie P1 — say
so. A proposal that a later run proved unnecessary gets closed with a note, not
silently dropped.

## Propose, don't self-modify

Present the proposals to the user in the closing message: what to change in
`SKILL.md`/references, why (pointing at what happened this run), and the version
bump it would imply. Apply them to the skill files **only with the user's
approval** — the skill's rules were approved once; changing them silently would
make every past approval meaningless.

If the user is absent (unattended run), leave the proposals in
`RETROSPECTIVES.md` marked `ÖNERİ — onay bekliyor` and surface them at the start
of the next attended run.

## Generalise before you propose — the skill stays project-agnostic

This skill must work unchanged on any project: frontend, backend or command-line
tool; any language, any framework, any domain. So `SKILL.md` and `references/`
never name a project, ticket, endpoint, table, framework-specific decorator or
domain concept — every lesson is admitted only as its generalised pattern.

**The litmus test:** *would this sentence be exactly as true in a different
repo?*

- "the X field of the Y endpoint rejects non-UUID values" — fails it.
- "a synthetic test value can fail validation before reaching business logic —
  separate the two rejections" — passes it.

What can't pass the test isn't skill material — it belongs in the **project's**
`.qa/` memory (known-issues, accepted-behaviours), which exists precisely to hold
project-specific knowledge.

Tool names are allowed only as per-ecosystem *menus with a selection rule* (as
`automation-toolbox.md` does), never as an assumed stack.

This includes the logs: `RETROSPECTIVES.md` and `CHANGELOG.md` identify a
motivating run only by date, level and surface type — never by project, ticket or
domain term. The run's full story belongs to that project's `.qa/`, which is
where anyone needing the detail should look.

## Versioning

The skill carries a semver `version` in the front-matter and a `CHANGELOG.md`
next to `SKILL.md`. On every **approved** change:

- **PATCH** (1.1.x): wording, clarification, reference-file edits that don't
  change behaviour.
- **MINOR** (1.x.0): a new rule, a new phase step, a changed default — anything
  that alters how a run behaves.
- **MAJOR** (x.0.0): restructuring the phase model or redefining verdict/level
  semantics.

Bump the front-matter version and add a dated `CHANGELOG.md` entry describing
what changed and **what motivated it** — the run's retrospective, or a review of
the skill itself. That trail is how "the skill is improving" stays a measurable
claim instead of a feeling.

## Release

This skill directory is a git repository (remote: `unalcakir28/qa-engineer-skill`,
private). Every approved version bump is released immediately:

```bash
git commit -m "vX.Y.Z — <one-line summary>"
git tag vX.Y.Z
git push && git push origin vX.Y.Z
```

`--follow-tags` skips lightweight tags — push the tag explicitly.

Standing permission for this exists for **this repository only** (granted
2026-08-20). It does not extend to any project repository, where commit and push
still require explicit user approval every time.

Retrospective entries without a version bump are committed and pushed too (no
tag), so the log never lives only on one machine.
