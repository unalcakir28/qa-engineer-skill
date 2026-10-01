# QA memory — the `.qa/` folder

Without memory, every run starts as a stranger to the project: the same bugs get
re-found, the same intended behaviours get re-reported as bugs, and nothing
accumulates. A few small markdown files in `.qa/` at the repo root fix that. They
are cheap to maintain (Phase 6, two minutes) and they compound.

```text
.qa/
├── README.md              # the index: file map + suite table — Phase 0 starts here
├── critical-flows.md      # never-break flows → tier C smoke, every run
├── known-issues.md        # every confirmed bug, forever (each tagged with Alan:)
├── accepted-behaviours.md # looks like a bug, is intended → don't report
├── regression-log.md      # one line per run; drives tier D rotation
├── environment.md         # env manifest: targets, dependency reality, access
├── metrics.md             # one metrics row per run; `escaped` fed by postmortems
├── contracts/             # public-contract snapshots for the breaking-change diff
├── evidence/              # raw request/response per case — NEVER committed
├── reports/               # run reports: YYYY-MM-DD-<feature>.md (never at .qa root)
└── suites/
    └── <feature>.md       # case lists, each with a unique ID prefix (references/case-list.md)
```

**Scaling rules — cheap now, expensive later:**

- **`README.md` is the index, not documentation.** One line per file, plus the
  suite table: suite → prefix → area covered → case count → last run + verdict,
  and a short "uncovered areas" list (the tier D candidate pool). Update it in
  Phase 6; an index that drifts is worse than none. Flat files + a thin index
  beat deep folder hierarchies — this folder's main consumer greps.
- **Reports never live at the `.qa/` root** — always `reports/YYYY-MM-DD-<feature>.md`.
  One report per run lands here; the root must stay six core files.
- **Case IDs are suite-prefixed** (see `references/case-list.md`) so
  cross-references stay unambiguous as suites multiply.
- **Every `known-issues.md` entry carries an `Alan:` (area) tag.** The file grows
  forever by design; when it passes ~30–40 entries, split it by area into
  `known-issues/<area>.md` + an index — the tags make that split mechanical
  instead of archaeological.

**Version control:** `.qa/` is team memory — it belongs in git, like docs and
tests. The one exception is `evidence/`: raw exchanges can contain tokens and
PII, so add `.qa/evidence/` to `.gitignore` the first time you create it. And
no `.qa` file ever holds a credential — `environment.md` says *where* secrets
live, never what they are.

If `.qa/` doesn't exist, offer to create it on the first run: propose the files,
seed `critical-flows.md` from what the codebase obviously does (auth, payment,
the main business action), and let the user correct it. Don't create it silently
in someone's repo — ask once.

Keep these files short and factual. They are working memory, not documentation:
prune entries that stop being true.

**Suites grow AND shrink.** Tag each case in a suite as `core` (re-run every
time: bug-reproducers, money/permission/state cases, one representative per
equivalence class) or `swept` (passed clean twice on unchanged code → rotation
pool, re-run only when its area is tier A/B again or comes up in tier D). Merge
near-duplicates. The always-run core must stay executable in one sitting;
history lives in the file, not in re-execution.

---

## `.qa/critical-flows.md`

The flows that must never break. Smoke-tested every run (tier C) regardless of
what the diff touched.

```markdown
# Critical flows

## CF-1 — API key authentication
- Steps: GET /v1/... with a valid key → 200; invalid key → 401
- Why critical: every integration depends on it
- Last verified: 2026-08-20 ✅

## CF-2 — Order creation
- Steps: cart → POST /v1/orders → DB record + stock decrement
- Why critical: revenue flow
- Last verified: 2026-08-20 ✅
```

## `.qa/known-issues.md`

Every confirmed bug, forever. A bug found once is the cheapest bug to find
twice — before designing a matrix, check whether any of these could recur in the
area you're testing.

```markdown
# Known issues

## BUG-014 — Coupon applied twice on two concurrent requests
- Area: coupon / order creation
- Found: 2026-08-20 | Severity: S1 | Status: FIXED (commit abc1234)
- Root cause: no lock on the coupon usage counter
- Regression test: `tests/test_coupon.py::test_concurrent_redeem`
- Recurrence risk: any new feature with a coupon/quota/stock counter

## BUG-015 — Deleted record shows up in export
- Found: 2026-08-18 | Severity: S2 | Status: OPEN
- Note: the list query has a soft-delete filter, the export query doesn't
```

## `.qa/accepted-behaviours.md`

Things that look like bugs but are intended. This file is what stops the report
from crying wolf on the same three items every release.

```markdown
# Accepted behaviours (not bugs)

- We return 403 instead of 404 on purpose: a deliberate choice to prevent
  existence leakage. (Decision: 2026-07-02, Ünal)
- Dates are stored in UTC and converted in the UI; the API always returns UTC.
  Deliberate.
- An empty name field is accepted, so old integrations don't break.
```

## `.qa/regression-log.md`

One line per run. Gives you the rotation for tier D, and a history the user can
skim to see what has and hasn't been swept lately.

```markdown
# Test run log

| Date | Scope (tier A) | Rotation (tier D) | Scenarios | Findings | Verdict |
|-------|-----------------|-------------------|---------|-------|-------|
| 2026-08-20 | coupon discount | webhook processing | 47 | 1×S1, 2×S3 | NO-GO → fixed → GO |
| 2026-08-14 | invoice PDF | user invitations | 38 | 1×S2 | GO WITH RISK |
```

## `.qa/environment.md`

The persistent form of Phase 0's environment capability inventory. Read it at
the start of every run and verify only what might have changed; update it in
Phase 6 with what the run taught you. Never a credential in here — only where
credentials live.

```markdown
# Environment manifest

## Target environments
| Environment | Access | Prod? | Note |
|-------|--------|----------|-----|
| local | http://localhost:<port> | no | seed: <the project's seed command> |
| staging | https://staging.example.com | no | test users: secret manager `qa/staging` |
| prod | — | **YES — never tested** | only Phase 6.5 read-only smoke, on request |

## Access and authentication (test playbook)
<!-- Per surface: which method guards it + HOW to obtain the credential (script/seed/
     env name/vault path). The secret itself is NEVER written here. -->
| Surface | Method | Obtaining a credential | Note |
|-------|--------|-------------------|-----|
| <user API> | Bearer JWT | <token script / login flow> | test user: <who provides it / vault path> |
| <admin surface> | Basic auth | env: <VARIABLE_NAME> | |
| <partner API> | API token | <seed step / generation command> | scope: <...> |
| <webhooks> | signature / basic + IP filter | <how to fake it> | |

## External dependency reality
| Dependency | local | staging | Note |
|------------|-------|---------|-----|
| identity provider | real | real | |
| payment gateway | mock | sandbox | never real money |
| e-mail | none | sandbox | sending to a real address is forbidden |
| external RPC service | NONE | real | these cases are BLOCKED (environment) locally |

## Fixture inventory
<!-- A real dependency isn't enough: if the seed record a case needs doesn't
     exist, the case can't run — and that's a more common blocker than a missing
     dependency. Record what EXISTS and what's MISSING per entity; a missing one
     becomes BLOCKED (fixture) at design time, not discovered mid-run. -->
| Entity | Exists in the environment | Missing | How to create it |
|--------|------------------|-------|-----------------|
| <account / tenant> | <2; one is empty> | <a member user with the second role> | <seed command / API call> |
| <balance / quota> | <zero balance only> | <an account with a positive balance> | <script> |
| <state machine record> | <draft, paid> | <expired, refunded> | <how to drive it to that state> |

## Known environment traps
- <e.g. local timezone is UTC, staging isn't>
```

## `.qa/metrics.md`

One row per run, appended in Phase 6. This is the skill's own scorecard — the
`escaped` column is filled retroactively by the postmortem loop and is the most
honest number in the table. Trends matter more than any single row: fake FAILs
and false alarms trending down means the process rules are working; `escaped`
staying at zero is the only claim that ages well.

```markdown
# Run metrics

| Date | Feature | Level | Cases | Findings (S1/S2/S3/S4) | Findings/case | False alarms (ruled out in Ph3) | Fake FAILs (harness) | BLOCKED | Escaped | Cost (~) |
|-------|---------|--------|------|---------------------|------------|---------------------------|----------------------|---------|-----------------|-------------|
| 2026-08-20 | coupon discount | L3 | 120 | 0/1/2/0 | 0.025 | 1 | 3 | 1 | 0 | ~3 hours |
```

## `.qa/contracts/` and `.qa/evidence/`

- **contracts/** — one snapshot per public contract (OpenAPI JSON, schema dump,
  proto). Phase 0 diffs the live contract against it; breaking changes become
  tier A cases. Refresh only in Phase 6, after the verdict.
- **evidence/** — `<YYYY-MM-DD>-<feature>/<case-id>.*` raw exchanges for FAILs,
  reproduced findings and sampled PASSes. Git-ignored, pruned only with user
  consent.

---

## How to use it in a run

- **Phase 0:** read all four, plus any existing `suites/<feature>.md` for the area
  you're about to test. Critical flows become tier C cases; known issues
  become candidate recurrence cases; accepted behaviours become a do-not-report
  list; the log tells you which area to rotate into tier D. `environment.md`
  replaces rediscovering the environment; the `contracts/` diff seeds
  breaking-change cases.
- **Phase 1:** extend the existing case list rather than writing a new one.
- **Phase 3:** before promoting a finding, check it against
  `accepted-behaviours.md`.
- **Phase 6:** append the new bugs, log the run in both `regression-log.md` and the
  suite's own `## Run history`, add any new critical flow, record anything the
  user just declared intended.

One more compounding move worth suggesting to the user: every regression test
written during a run should be committed and wired into CI. The `.qa/` memory
keeps *you* from repeating work; CI keeps the *codebase* from regressing between
runs. Together they're the difference between a QA session and a QA practice.
