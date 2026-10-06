# qa-engineer — Changelog

Semver. Every approved change is recorded here, along with which run's
retrospective it came from. Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.14.0] — 2026-10-06

Motivation: retrospective of the 2026-10-06 L3 run (backend API, a new read-only
aggregation endpoint plus new filters on an existing list, fix mode on). The most
serious defect attributable to the change was additive on the wire but would have
silently shifted existing calls in the client generated from the contract; the
contract diff had classified it as "additions only". One false finding came from
a QA bootstrap: with the credential check bypassed, a malformed credential reached
code the real build never lets it reach and produced a 5xx.

### Added
- `SKILL.md`, contract diff: additive on the wire is not additive for a generated
  client — check parameter order, derived method names and new enum/union names;
  a new optional input goes after the existing ones.
- `SKILL.md`, access playbook: a defect on a path the bootstrap altered is not a
  finding until it reproduces on a build without the bootstrap.
- `SKILL.md`, Phase 3: a harness bypass on the path is listed among the
  alternative explanations.
- `references/backend-api.md`: generated-client parameter order under
  Compatibility.

## [1.13.0] — 2026-10-05

Motivation: retrospective of the 2026-10-05 L2 run (backend API, a new
alternative input for a scheduling rule, fix mode on). Both defects attributable
to the change came from the cold case-list review, not from execution. One was an
accepted value under which the rule never applied. The other was a missing upper
bound that a sibling input had. In both, the code did exactly what its author
meant. One false FAIL came from an executor computing a money delta itself.

### Added
- `references/oracles.md`: the **Effect** oracle. Every accepted input value must
  change behaviour. An accepted value that makes the feature a no-op, or freezes
  state indefinitely, is a defect. A bound present on a sibling input and absent
  on the new one is a lead.
- `SKILL.md`, cold review: the reviewer is pointed at inputs that are accepted but
  have no effect, or have no upper bound.
- `references/case-list.md`: money, quantity and counter deltas are written as the
  expected number, never as the rule for computing it.

## [1.12.0] — 2026-10-05

Motivation: retrospective of the 2026-10-05 L3 run (backend API, a new
per-entity attribute written on the reward load path, fix mode on). The A/B
differential found a regression no response showed: a new write path moved a
column the persistence layer stamps on its own, which reports sort and filter
on. A hand-written query bound a time value whose meaning depended on the
database session's time zone. Two expectations copied from REST convention
produced false FAILs.

### Added

- `techniques.md` §11: in an A/B run, read back persistence-maintained columns
  (auto-stamped update times, version counters, trigger fields) on both builds,
  not only the responses.
- `oracles.md`: the project's own error and response conventions outrank
  textbook ones when writing an expectation; a deviation from the textbook alone
  is an open question, not a FAIL.
- `edge-data.md`: run a hand-written query that binds a time value under two or
  more database session time zones.

## [1.11.0] — 2026-10-02

Motivation: retrospective of the 2026-10-02 L2 run (backend API, idempotency
keys and asynchronous batch verdicts, fix mode on). The independent design
review added nine cases, and the two defects the run found came from them; the
A/B differential had to run the builds sequentially because both consumed the
same queues; and a dependency-lookup stub plus a credential-check bypass made
otherwise blocked cases executable.

### Added

- Phase 1: an independent cold review of the case list is the default at L2 and
  above (mandatory when you wrote the change), with review-added cases marked.
- Phase 0: replace a lookup-only absent dependency with a harness-driven stub
  (with failure-mode switches) instead of blocking every case behind it.
- Phase 0 access playbook: the "disable only the credential check" fallback,
  with its reporting obligation (closes the P3 proposal from 2026-10-01).

### Changed

- `techniques.md` §11: a third A/B hygiene rule — builds that share a queue,
  broker, outbox or scheduler run one at a time (or with separate namespaces)
  for any probe decided off the request path.

## [1.10.0] — 2026-10-01

Motivation: retrospective of the 2026-10-01 L3 run (backend API, a permission
restriction on a state-mutating endpoint). Every case shared one seeded graph
of related records, and the concurrency cases deliberately drained the same
shared counters, so splitting execution across parallel agents would
have made them corrupt each other's assertions.

### Changed

- **`references/run-modes.md` — the lead may execute a large list itself** when
  the groups cannot be given isolated fixtures, provided the cases are
  mechanised as scripts that record status and raw evidence per case. The
  deviation and its reason go into the report and the metrics row. Delegation is
  a means (context protection, cheaper tokens), not a goal.

## [1.9.0] — 2026-09-22

Motivation: retrospective of the 2026-09-22 L3 run (backend service layer,
a concurrency/locking fix; the project's existing mock-based suite was fully
green while carrying zero information about the change's actual claim).

### Added

- **Non-negotiable #9 — a test must run at the level its claim lives at.** If a
  change makes a claim about concurrency, locking, isolation, transactions or
  ordering, the existing test suite is not coverage of that claim however green
  it is: test doubles don't hold locks, don't enforce an isolation level, don't
  produce a commit order. Such a change must run at least one case against the
  real dependency (a real database, a real broker, real parallel processes); if
  it can't, the report states in those words that the claim is untested.
- **Phase 3 — a timing finding is not a number until benign orderings are
  filtered out.** Not every round of a parallel run is "bad"; a round where the
  condition simply arrived after everything else is expected behaviour. For a
  round to count as a finding, it must be shown that some actor genuinely
  observed the new state. One finding first measured 13 out of 25; after
  filtering, the real number was 3 out of 30 — same defect, one tenth the claim.

### Changed

- **Non-negotiable #3 (red-green) extended for concurrency.** For a defect that
  only appears under concurrency, timing or load, a green result carries no
  information until the same harness has been shown to go red against the
  unfixed code — otherwise green is indistinguishable from "the scenario never
  actually occurred".
- **`references/techniques.md` §11 (A/B differential)** now spells out the
  technique's second function explicitly: **harness validity**, as much as
  attribution. For concurrency claims, the baseline build going red is the
  license to trust the new build going green.
- **Catalogue row 9 (concurrency & idempotency)** now requires at least one case
  to run against the real dependency, and references #9.

## [1.8.0] — 2026-09-16

Motivation: retrospective of the 2026-09-16 L3 run (backend API, a security
boundary fix; the person running it was also the person who wrote the change).

### Added

- **`references/techniques.md` §11 — differential (A/B) execution.** The
  pre-change build is stood up alongside the current build against the **same**
  data store, and every probe requiring attribution is run twice. "Did I create
  this finding myself" stops being a debate and becomes a measurement; a
  byte-for-byte comparison of surfaces that shouldn't have changed gives an
  unplanned regression sweep for free; and since the baseline is already the
  pre-fix code, red-green evidence comes for free too. Proving the baseline's
  validity (which commit / which point in time) **before** trusting the first
  result is part of the rule — otherwise every "pre-existing" label is
  unsupported.
- **Phase 0 — fixture inventory.** The twin of the dependency inventory: per
  entity, which seed record exists in the environment and which doesn't. A
  missing one becomes `BLOCKED (fixture)` at design time. Its permanent home is
  `.qa/environment.md` (template in `qa-memory.md`). Rationale: in that run,
  almost every `BLOCKED` case was a **data** gap, not an environment gap, and
  all of them were discovered mid-run.
- **Phase 0 — the "can the baseline build even run" question.** The A/B
  decision is made in Phase 0, because it affects both the case list (every
  attribution-requiring probe runs twice) and the verdict; if it doesn't apply,
  the reason is stated.

### Changed

- **Attribution dimension added to the verdict rules (`release-gate.md`).** The
  table said "open S1/S2" without asking *whose*. Applied literally, a change
  that fixes a serious defect gets blocked by an unrelated defect it merely
  walked past — leaving **both** in production. The verdict is now computed on
  findings **attributable to the change**: a finding verified as pre-existing
  via A/B does not produce `NO-GO`, it caps the verdict at `GO WITH RISK` and
  gets its own ticket. Two exceptions put it back on the change's account: the
  change was supposed to fix it, or the change makes it more reachable / more
  severe. Only a finding that appears solely on the new build is a full-weight
  regression.
- **Phase 3 — "pre-existing" is a measurement, not a hunch.** Since this label
  now changes the verdict, it is held to the same evidence standard as the
  finding itself: if a baseline exists, the probe is run against it too and
  both outputs are attached; if not, the label is written as *likely
  pre-existing*.
- **Design independence added to non-negotiable #6.** "Verify your own fix with
  fresh eyes" covered execution but not design: whoever wrote the change also
  passes their own blind spots on to the case list — they cannot design a case
  for something they never thought of. Cases for newly written code must be
  designed or reviewed by something other than the mind that wrote it.
- **`test-data.md` — "read the value, not the name".** The escape hatch from
  the synthetic-value rule's format check: for anything whose definition lives
  elsewhere (an enum member, a scope, a status code), the **value** is read
  from that definition rather than re-derived from the identifier's name in
  code. A member named `READ_ONLY` whose actual value is `read:only` fails
  validation just like a typo would, and is indistinguishable from a real bug.
  That run traced two fake FAILs back to exactly this.

## [1.7.0] — 2026-09-14

Motivation: not a run retrospective — a review of the skill itself (at the
user's request, a structure and cost audit).

### Added

- **`references/cli-tool.md`:** a third surface checklist — command-line tools
  and binaries. The contract is defined as exit code + stdout/stderr + what it
  did to the disk; covers argument parsing, hostile filesystem input (symlink
  cycles, sparse files, NFC/NFD, permission errors, dead mounts), signals and
  cancellation, stream/pipe/TTY behaviour, config precedence, privilege
  boundaries, cross-platform behaviour and packaging. Previously only
  `backend-api` and `web-frontend` existed — the CLI surface was entirely out
  of scope.
- **The "surface decides the category" rule (Phase 1):** if a category doesn't
  apply, it is stated once with the reason; never silently skipped, and no
  empty cases are generated either.

### Changed

- **Phase 6.5 was in the wrong place:** it appeared *before* Phase 6 in the
  file. Moved to after Phase 6 and trimmed to three lines pointing at
  `release-gate.md`.
- **Cold sections moved into `references/`** — SKILL.md went from 7,773 to
  6,851 words (the cost loaded on every trigger; ~1,450 words moved out, ~250
  words of pointers and ~130 words of new rules added in their place). Moved:
  PR/Sentinel run modes and parallel-run rules → `run-modes.md`; the escaped-bug
  postmortem loop → `postmortem.md`; retrospective entry format, the
  generalisation test, semver and the release procedure → `skill-maintenance.md`.
  Each has a one-line pointer in place; none of it was on a normal run's hot
  path.
- **`description` 1,008 → 797 characters.** Only 16 characters of headroom were
  left before the 1,024 limit; redundant Turkish trigger synonyms ("test yap",
  "hata bulmaya calis", "canliya cikmadan once kontrol et") were trimmed, and
  "CLI command" was added.

## [1.6.0] — 2026-08-20

Motivation: a `.qa/` scaling review with the user — finding things without
getting lost as the folder grows.

### Added

- **`.qa/README.md` index:** file map + suite table (prefix, area covered, case
  count, last run, verdict) + an uncovered-areas list (the tier D pool); kept
  current in Phase 6. Principle: flat files + a thin index beat a deep folder
  hierarchy — this folder's main consumer greps.
- **Suite-prefixed case IDs:** every suite file declares a unique short slug at
  the top (`KPN-001`), a bare `TC-` is forbidden — prevents cross-reference
  ambiguity as suites multiply. Template and examples updated.
- **`.qa/reports/` folder:** reports go here as `YYYY-MM-DD-<feature>.md`; the
  `.qa` root stays fixed at six core files.
- **known-issues `Area:` tag:** the log grows forever by design; past ~30-40
  entries, splitting by area becomes mechanical thanks to the tags.

## [1.5.0] — 2026-08-20

Motivation: user directive — project-dependent content in retro notes must not
stay in the skill repo.

### Changed

- **Retro is now a proposals ledger, not a run diary (Phase 7):** an entry is
  identified only by date + level + surface type; content is generalised
  lessons + proposals. A lesson that can't be generalised goes to the
  project's `.qa/` instead. The provenance exception was REMOVED — logs may no
  longer contain a project/ticket/domain term either.
- Existing `RETROSPECTIVES.md` and `CHANGELOG.md` entries were anonymised under
  this rule.

## [1.4.0] — 2026-08-20

Motivation: user directive — "every time I say 'test this', don't rediscover
how to test it; pull what you can from the project, ask once for what you
can't and save it project-scoped, update it when it changes."

### Added

- **Access playbook (Phase 0):** an authentication section in
  `environment.md` — per-surface method + how to obtain the credential
  (script/seed/env name/vault path; the secret itself is never written). Strict
  order: (1) if documented, use it as-is, don't rediscover it; (2) if not,
  derive it from the project (guards, auth config, existing scripts, the
  project's own tests); (3) if it can't be derived, ask the user ONCE and write
  the answer into the manifest. Asking a documented fact a second time is a
  process failure → logged in the retrospective. If the method
  changes/breaks/a new one appears, the manifest is updated in Phase 6.
- **Git release protocol:** the skill directory became a git repository
  (`unalcakir28/qa-engineer-skill`, private). Every approved version bump =
  commit + `vX.Y.Z` tag + push (standing permission for this repo only, granted
  2026-08-20; does not cover project repositories). Retro entries are committed
  without a tag.

## [1.3.0] — 2026-08-20

Motivation: an improvement session with the user — 8 proposals discussed, all
approved.

### Added

- **Escaped-bug postmortem loop:** for every bug that escaped to production —
  attribution (which run, which category) → miss diagnosis (design gap / false
  PASS / parked / out of scope) → regression case + known-issues entry +
  `metrics` `escaped` increment → an automatic P1 proposal if it generalises.
- **`.qa/environment.md` environment manifest:** the Phase 0 inventory is now
  persistent — read, verified, updated in Phase 6; a credential is never
  written into it.
- **`.qa/metrics.md` effectiveness metrics:** findings/case, false alarms, fake
  FAILs, BLOCKED count, escaped bugs, cost per run — a numeric answer to "are we
  improving".
- **`.qa/evidence/` evidence archive:** raw request/response for FAILs,
  reproductions and sampled PASSes, one file per case ID; **never committed**
  (added to `.gitignore`), pruned only with the user's consent.
- **`.qa/contracts/` contract diff:** a public-contract snapshot diffed every
  run; a breaking change automatically becomes a tier A case and a candidate
  finding.
- **`references/test-data.md`:** the recipe for fixture isolation — prefix
  schemes, isolation levels, idempotent seeding, synthetic-value validity (test
  the business rule, not validation), time boundaries, cleanup, secrets rules.
- **PR mode:** tier A = the PR diff; a condensed PR-comment output in addition
  to the standard report; posting to the PR only with explicit approval.
- **Sentinel mode:** a definition for scheduled/CI runs — L2 report-only, tier
  C smoke + tier D rotation + tier A for whatever changed since the last run;
  unattended never fixes code, never prunes data, never sends anything
  outward-facing.
- **`.qa/` version-control rule (qa-memory.md):** `.qa/` is committed (team
  memory), `.qa/evidence/` is ignored, no `.qa` file ever holds a secret.

## [1.2.0] — 2026-08-20

Motivation: user directive — the skill must work unchanged on any project
(frontend/backend, any stack).

### Added

- **Project-independence rule (Phase 7):** `SKILL.md` and `references/` never
  contain a project/ticket/endpoint/framework-decorator/domain concept; every
  lesson is admitted only as a generalised pattern (litmus test: "would this
  sentence be exactly as true in a different repo?"). Content that can't be
  generalised goes to the project's `.qa/` memory instead. Tool names may only
  appear as a per-ecosystem menu with a selection rule. Logs (CHANGELOG/
  RETROSPECTIVES) are the provenance exception.

### Audit note

- As of v1.2.0, `SKILL.md` + `references/` were scanned: no project-specific
  content found. `automation-toolbox.md` (the ecosystem menu) and `oracles.md`
  (the peer-API example) were found to comply with the rule.

## [1.1.0] — 2026-08-20

Motivation: retrospective of the first real field run (2026-08-20, L3, backend
API, 120 cases, GO). See `RETROSPECTIVES.md` → 2026-08-20.

### Added

- **Phase 7 — Skill retrospective:** at the end of every run, the skill
  evaluates itself, writes an entry to `RETROSPECTIVES.md`, and presents
  improvement proposals to the user; skill files change only with approval.
- **Versioning:** a front-matter `version` field + this CHANGELOG; PATCH/MINOR/
  MAJOR rules defined in Phase 7.
- **Fixture isolation rule (Phase 2):** every case that mutates state creates
  or restores its own fixture; parallel agents never share data, and a group
  touching shared state runs alone. (This run had produced 3 fake FAILs.)
- **"Suspect your own harness first" reflex (Phase 2):** on an unexpected
  cluster of FAILs, the response body is logged and validation rejections are
  separated from business rejections. (In that run, a synthetic value that
  failed format validation had returned a fake 400 across an entire case
  group.)
- **Subagent evidence contract + PASS sampling (Phase 2, parallel section):**
  raw request/response is mandatory; a PASS sample from high-risk categories is
  re-run in the main session; a sample that doesn't hold re-verifies that
  agent's entire group. (In that run, TC-099's FAIL turned out to be the
  subagent's own test error.)
- **Environment capability inventory (Phase 0):** external dependencies are
  marked real/mocked/absent at design time; cases needing an absent one get
  `BLOCKED (environment)` from the start. (In that run, the absence of an
  external RPC dependency was discovered mid-run.)
- **Cost tag on the level question (Kickoff):** every level option is presented
  with an estimated case count + time/token cost.
- **BLOCKED debt tracking (Phase 4 + 6):** a "to run in another environment"
  list in the report; carried into `regression-log.md` as input for the next
  run.
- **Suite pruning strategy (Phase 6):** cases are tagged core/swept; the
  always-run core stays small, clean-passing cases demote to the rotation pool,
  near-duplicate cases are merged.

### Notes

- The severity-rubric proposal was not applied: the rubric already existed in
  `references/reporting.md` (S1–S4 + Risk + Question).

## [1.0.0] — 2026-08-20

- Initial release: kickoff (level + fix/report), Phase 0–6.5, a 20-row category
  catalogue, the tier A–D scope model, the non-negotiable evidence rules, the
  model policy (design/verdict on Opus, execution on Sonnet), `.qa/` project
  memory, 10 supporting documents under `references/`.
