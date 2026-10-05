# qa-engineer — Proposals ledger (run retrospectives)

One entry at the end of every run (Phase 7). **This is a proposals ledger, not
a run diary:** a run is identified only by date + level + surface type;
no project, ticket, endpoint or domain term goes in here (the run's own story
lives in that project's `.qa/` folder). Proposals are rated P1 (caused a wrong
result) / P2 (significant time lost) / P3 (polish). A recurring proposal is
automatically promoted to P1. Applied proposals move into `CHANGELOG.md` and
are closed here as `APPLIED (vX.Y.Z)`.

---

## 2026-08-20 — L3, backend API

- **Cases:** 120 · **Result:** 117 PASS / 1 FAIL (S2, fixed) / 1 BLOCKED /
  1 SKIP · **Verdict:** GO · **Skill version:** 1.0.0

**Generalised lessons:**

- Catalogue row 8 (state & sequence), with the "re-trigger a closed record for
  the same reason" case, caught an S2 that improvisation would not have found;
  the red-green rule and the self-certification ban strengthened the fix's
  evidence.
- Scenarios that mutated state leaked fake FAILs into later cases without
  fixture isolation (3 instances).
- A synthetic value that failed format validation caused an entire case group
  to be rejected before it ever reached the business rule — it was one step
  from being reported as a code bug.
- One subagent FAIL was caused by the subagent's own malformed request; it was
  indistinguishable without the raw request/response.
- An external dependency's absence from the target environment was discovered
  mid-run — wasted effort on the affected cases.

**Proposals:**

- P1 — fixture isolation rule → APPLIED (v1.1.0)
- P1 — harness-first suspicion reflex → APPLIED (v1.1.0)
- P2 — subagent raw evidence + PASS sampling → APPLIED (v1.1.0)
- P2 — Phase 0 environment capability inventory → APPLIED (v1.1.0)
- P2 — cost tag on level options → APPLIED (v1.1.0)
- P3 — BLOCKED debt tracking → APPLIED (v1.1.0)
- P3 — suite pruning strategy → APPLIED (v1.1.0)
- P3 — severity rubric proposal CLOSED: the rubric already existed in
  `references/reporting.md`; the problem was discoverability, and Phase 4
  already points there.

**Open proposal (awaiting approval):** none.

---

## 2026-08-21 · L3 · backend API (money flow + scheduled job + escrow accounting)

**Generalised lessons:**

- **The assumption that an endpoint returning HTTP 2xx means the work is done is
  the most productive fake-FAIL generator there is.** When a write request hands
  the work off to a queue, the response signals "accepted", not persisted state;
  a balance/status read immediately after races the worker. In this run alone it
  produced 3 phantom FAILs, one step away from being reported as product bugs.
  Rule candidate: *in a status-changing asynchronous flow, waiting for the
  transition to a terminal state before asserting is mandatory; without that
  wait, the case design is incomplete.*
- **A money comparison cannot use the language's default numeric type.** A float
  subtraction on decimal balances (`100000 - 99999.01`) produced an artificial
  cent-level mismatch, ready to be reported as "MISMATCH". Rule candidate:
  *money invariants are compared only in decimal/rational type (or integer
  cents).*
- **A shared harness silently corrupts multiple agents' evidence with a single
  wrong path.** A wrong endpoint path was returning 404; a permission case that
  mistook 404 for "access denied" could have produced a fake PASS. Two agents
  caught it independently. Rule candidate: *every endpoint path in a shared
  harness must be verified on first use with a "does the path exist" check
  (404 ≠ 401/403); a path fix must be announced to every agent running.*
- **A case's own formula can itself be a basis error.** I had written an
  accounting invariant that mixed up net and gross; the agent reported "FAIL"
  and diagnosed it correctly itself (the amount now exactly equalled the
  refunded total). The system was right, the spec was wrong. A rule candidate
  already exists (*the basis itself can be defective*), but it is specifically
  weak for **numeric invariants**: writing a protective invariant requires
  separating the net flow from the gross flow.
- **Cases that need a long wait (cron/periodic job) block the agent and delay
  the report.** Two agents stalled on their own background wait; results only
  came back after a "report what you have" message. Meanwhile a deterministic
  equivalent of the same feature (dropping the job directly on the queue, or
  checking the trigger condition's trace in the DB) took seconds. Rule
  candidate: *for a time-dependent mechanism, run the deterministic equivalent
  first, and wait for the real scheduler only once, for an end-to-end check;
  group waiting cases separately so the report doesn't get stuck on them.*
- **An agent's "FAIL" can also come from the case's expectation being stricter
  than the code's documented contract.** The security behaviour was correct (no
  credit, a loud error, a DLQ alarm); only the diagnostic fields I expected
  weren't populated. Phase 3 verification filtered this out before it became a
  finding.
- **Parallel agents' direct DB mutations show up as a "violation" in another
  agent's global invariant sweep.** A row that one agent deliberately corrupted
  tripped another agent's integrity sweep into a FAIL; the `referenceId` prefix
  made it traceable. Rule candidate: *a case running a global invariant sweep
  must attribute a found violation to a fixture prefix when it finds one; if the
  owner is another agent, it is not a product finding.*

**Proposals:**

- **P1 — a mandatory-wait rule for the async-acceptance (202/201-then-worker)
  pattern.** Into `SKILL.md` Phase 2 and `references/backend-api.md`: if a write
  request hands the work off to a queue/worker, waiting for the transition to a
  terminal state before asserting is part of the case design. (3 phantom FAILs
  in this run; if it recurs in a future run, the "harness-first suspicion" rule
  catches it but at high cost.)
- **P1 — decimal-arithmetic requirement for money invariants.** A sentence added
  to `SKILL.md`'s "Assert values, not vibes" item: money/rate comparisons are
  never done in the language's float type. (1 phantom finding in this run.)
- **P2 — endpoint-path verification reflex in a shared harness.** In
  `references/test-data.md` or `backend-api.md`: confirm a path exists via the
  404/401 distinction; announce a shared-harness path fix to every agent
  running.
- **P2 — a "deterministic equivalent first" rule for time-dependent cases**, and
  grouping waiting cases separately (`SKILL.md` parallelisation section).
- **P3 — violation ownership in global invariant sweeps.** A violation found
  during a parallel run doesn't count as a finding until attributed to a
  fixture prefix.

**Open proposal (awaiting approval):** the P1/P1/P2/P2/P3 above — awaiting
user approval. If approved, a MINOR version bump (new rules change behaviour):
v1.7.0.

---

## 2026-08-27 · L3 · backend API (visibility / information-disclosure boundary)

**Caught thanks to structure:**

- **A contract sweep found what two review rounds missed.** The feature had
  gone through two separate code-review rounds; what caught one of the two
  S3s found (an internal flag leaking into the client contract) was running
  the exploratory charter as "scan the schema systematically": programmatically
  extracting **every** endpoint carrying the target field from the published
  API schema. Trying endpoints one by one didn't find it; the schema graph did.
- **Sibling-endpoint comparison was the second S3's oracle.** "Three endpoints
  in the same family return 400 for this input, this one returns 500" — the
  strongest evidence anchoring a rule that wasn't written in the spec.
- **The "suspect your harness" rule cut off a false alarm before it was
  reported.** What looked like a permission bypass was caused by the elevated
  role the test itself had granted. Reading the guard code saved it from being
  reported as an IDOR.

**Time/noise cost:**

- **A build step killed a running watch process.** Running an "check
  everything" command mid-verification-pass rewrote the build output the watch
  process was reading; the app couldn't find its module and crashed, and every
  verification request returned a connection error. One round wasted.
- **A fee/balance precondition left three cases BLOCKED on the agent side.**
  Write-path cases were blocked by the rule against touching the shared
  fixture; once the main session set up the precondition, all three ran in
  seconds. The fixture's "write-path preconditions" should have been set up
  from the start.

**Things I improvised that the skill should say:**

- Writing fixture setup **through the product's own ORM** (instead of raw
  SQL): it surfaces required-field/relationship errors readably and speeds up
  iteration.
- Writing the distinction explicitly into the agent brief: a read-only shared
  fixture versus a mutating case's obligation to **create its own isolated
  copy**.
- **Keeping cases that need global state in the main session** (e.g. "there is
  no X anywhere in the system"): no parallel agent can run them, because they
  poison the others.

**Non-negotiable strained:** the "re-run every FAIL yourself" rule was the
highest-payoff rule in this run — two of three agent findings were confirmed,
one (permission) was refuted, and an S1 claim carried over from an earlier
run turned out to be empirically wrong. The rule is expensive but
indispensable.

**Proposals:**

- **P1 — a "scan the contract schema programmatically" step should be a
  mandatory sub-item of exploratory charters.** Into `SKILL.md` Phase 1
  category 15 and `references/backend-api.md`: if a feature is about hiding a
  field/record, extract every endpoint carrying that field from the published
  schema and turn each into a case. Counting endpoints by hand misses this
  class.
- **P1 — a rule comparing the published contract against the runtime
  response.** Into the same two reference files: the declared schema's field
  set must be compared against the actual response body's field set; an extra
  field (returning the raw entity) is a finding — a new column enters the
  contract the day it's added.
- **P2 — a "don't run a build command while a watch process is up" warning.**
  Into `references/backend-api.md` or the environment manifest template: watch
  processes sharing build output crash when a full build runs; only tests +
  type checking run during a verification pass.
- **P2 — a fixture write-path-precondition checklist.** `references/test-data.md`:
  the fixture must set up not only read cases but write cases' preconditions
  too (fee balance, quota, permission record); otherwise agents report them
  BLOCKED and the main session does the same work a second time.
- **P3 — exempting cases that need global state from parallelisation.** Into
  `SKILL.md`'s parallelisation section: "there is no X anywhere in the system"
  type cases stay in the main session, never handed to an agent group.

**Open proposal (awaiting approval):** the P1/P1/P2/P2/P3 above. The previous
run's proposals are also still awaiting approval; **a recurring pattern:**
rules that reduce "harness-caused fake FAIL" came out P1 in two runs in a
row — a sign they've earned approval. If approved, MINOR: v1.7.0.

---

## 2026-08-27 · L2 · backend API (CQRS, campaign/reward area)

A change that had already been through two review rounds (one refactor, one
static-analysis review + fix) was tested. **Neither** of those two rounds had
seen any of the four findings that came out of this run; two came from
entirely outside the diff.

**Caught thanks to structure:**
- **Category 13 (mass-assignment probe) alone found an S1:** authorization was
  keyed on a URL parameter, while a same-named field in the body won,
  producing a write that bypassed the permission check. It had nothing to do
  with the feature itself — without the catalogue row forcing it, it wouldn't
  have been tested.
- **Category 9's "cross-user parallel" distinction** found an S2: a shared
  budget cap held for the same user but not across different users. Reading
  "concurrency" as "same record" and moving on misses this class.
- **Category 19 + the target environment being behind:** the target
  environment hadn't applied the migration yet; this was the only window to
  test the data-transformation path against real data.

**Where it cost time/noise:**
- I didn't feed my own environment hints (the listing endpoint requiring
  coordinates) into my own measurement → a fake FAIL. The agent instructions
  were better informed than the main session.
- The full build command killed a running watch process (one round lost) —
  this was already the previous run's P2 proposal, and it recurred because
  that proposal wasn't approved.

**Things I improvised that should be rules:**
1. **A security finding's impact isn't measured at the first barrier.** The
   write attempt against an unauthorized target got a 400 from a secondary
   control (resource-ownership check); stopping there would have said "limited
   impact". When a second path that bypassed that control (a resource shared
   across every tenant) was tried, the finding rose from S2 to **S1**. Rule:
   for a bypass finding, the secondary control that produced the first
   rejection must itself be attacked before severity is assigned.
2. **A data-transforming migration can only be tested before it's applied.**
   The data to be transformed must be created BEFORE the migration; in an
   environment where it's already applied, that path can no longer be
   observed, and "the migration ran" proves only the schema change, not the
   transformation.
3. **An environment hint written into the agent instructions applies to the
   main session's own run too.** If the same hint isn't written in both
   places, the main session produces its own fake FAIL.
4. **A fix without a regression test is an incomplete fix.** A controller-level
   ordering fix had no regression test in the project; writing the test and
   temporarily reverting the fix to prove it went red (not just seeing
   "green") showed the test was actually asserting something.

**Non-negotiable observation:** #3 (red-green) was strained on a controller
fix because the project had no test pattern for that layer. Establishing the
pattern instead of bending the rule turned out to be the right call — but the
skill doesn't say "if the fix's layer has no test pattern, establishing one is
part of the fix."

**Proposals:**
- **P1 — a rule to also test the secondary control on bypass findings** (item
  1 above). Into `SKILL.md` Phase 3: severity is assigned only after a second
  path bypassing that barrier is attempted, not by stopping behind the first
  barrier.
- **P1 — a test window for data-transforming migrations** (item 2 above).
  `SKILL.md` category 19 and `references/backend-api.md`: a transformation
  case is set up before the migration is applied; if the environment has
  already applied it, that case is written as `BLOCKED`, not `PASS`.
- **P2 — a single source for environment hints** (item 3 above). Every
  environment hint entered into an agent instruction is written to the
  environment manifest at the same time; the main session reads its own cases
  from that manifest.
- **P2 — if the fix's layer has no test pattern, establishing one is part of
  the fix** (item 4 above). Into `SKILL.md` Phase 5.

**Open proposal (awaiting approval):** the P1/P1/P2/P2 above + the previous
two runs' still-pending proposals. **A pattern recurring for the third time:**
the "don't run a full build while a watch process is up" warning actually cost
time in this run — it has stood as P2 for three runs now and should count as
P1. If approved, MINOR: v1.7.0.

---

## 2026-09-01 — L3, backend API (second entry point / tool surface)

- **Cases:** 231 · **Result:** 214 PASS / 10 FAIL (6× S2, 7× S3, 5× S4) / 3 open
  questions / 7 NOT RUN-BLOCKED · **Verdict:** NO-GO · **Skill version:** 1.6.0 ·
  **Execution:** 7 parallel Sonnet groups + lead verification

**Generalised lessons:**

- **Catalogue row 15 (error guessing / exploratory) alone produced the run's
  three most serious findings** (6 cases → 3 FAILs, all S2). The categories the
  matrix could produce verified the surface's contract; only timeboxed
  exploration found where the contract was *internally inconsistent*. The
  clearest evidence yet that row 15 deserves its "always" mark.
- **A boundary/guard that is safe on a read path backfires on a write path.**
  Rejecting the response means "gave an incomplete answer" on read; on write it
  produces "the side effect happened but was reported as not having happened"
  — the caller retries and a duplicate record is created. This was the run's
  most serious finding, and no catalogue row asks this directly.
- **When a feature adds a second entry point to existing logic, the protections
  the two entry points DECLARE must be diffed at the source level.** Black-box
  testing answers "was it rejected", not "was the right protection chosen".
  This diff produced a privilege-escalation finding (S2): a body field was
  triggering a lifecycle transition that had its own separately defined
  permission, using a weaker permission instead.
- **If two masking/protection surfaces exist for the same data class, one of
  them is missing something.** A disagreement between the two surfaces is the
  finding itself and ends the debate: the "is this intentional" question is
  answered by the project masking the same value elsewhere.
- **If two inputs of the same class are validated at different times, the one
  validated later silently escapes into production.** One type reference was
  rejected at create time while its sibling only failed at execution time — and
  the dry-run validation tool reported clean for the second one.
- **Fixture isolation in parallel execution must NOT be limited to data:
  identity and authorization objects must be isolated too.** Two groups had
  been assigned in a way that mutated the same permission group; a mid-run
  correction message had to be sent. The fake-FAIL risk arises exactly in the
  category (security/permissions) where it's most expensive.
- **Every shared artifact named in a brief given to executors must be verified
  as actually being written BEFORE dispatch.** A file the brief said "every
  call accumulates into this file" was never written to by the harness at all;
  one agent spent significant effort manually rebuilding a profile.
- **Setting up a shared harness that automatically archives evidence per
  case-id before dispatch turned the "raw evidence for every case" rule from a
  discipline into a free side effect.** Across 7 groups, not a single "no
  evidence" situation came up.
- **My own case list produced 3 fake FAILs:** a wrong constant-value
  expectation, an assumption of a schema column that didn't exist, and a script
  that swallowed a setup step's error. All three died in Phase 3 — but all
  three could have been prevented during case design with a single "where did
  I get this expectation from" question.
- **The pressure to report a missing feature as a bug is real:** the absence of
  a uniqueness rule that was never promised anywhere was reported by one
  executor as "contradicts the docs". The "no basis, no finding — it's an open
  question" rule held the line.

**Proposals:**

- **P1 — extend the fixture isolation rule to identity/authorization objects.**
  SKILL.md's parallel-execution section's "each agent seeds its own isolated
  fixture set" item talks about data only. Authorization objects — identity,
  role, permission group, API key — need the same rule; otherwise two groups
  mutate each other's permission state and the fake FAIL lands exactly in the
  security category. (A mid-run correction was needed in this run.)
- **P1 — a new heuristic in `references/oracles.md`: "second entry point
  diff".** When a feature adds a second entry point (tool surface, queue
  consumer, batch job, admin CLI) to existing logic, compare that entry
  point's declared protections against the first's declarations at the source
  level. Black box sees "was it rejected", not "was it the right protection".
  This produced an S2 in this run.
- **P2 — into `references/oracles.md`: "a boundary validated on the read path
  must be re-tested on the write path".** Guard/limit/cutoff behaviours must be
  tested together with the question of whether they run before or after the
  side effect; rejecting after the side effect produces "silently completed but
  reported as failed". This run's most serious finding.
- **P2 — a line in Phase 2: verify the shared harness and every shared artifact
  named in the brief with one call before parallel dispatch.** A file that's
  named but never written means measurable wasted effort per agent.
- **P3 — a named section in the report template for "suites not run because
  they'd mutate shared state".** Improvised as §6 in this run; not the same
  thing as `BLOCKED (environment)` (the environment isn't missing, running it
  would break *something else*) and it belongs on the carried-forward debt
  list for a different reason.

### Same run's fix phase (same day, 18 fixes)

- **Verifying a fix with only a unit test can hide that the fix never worked at
  all.** A fix turned out to be a no-op on a path where the same data arrived
  in two different CLR shapes (a value set in memory vs. a value returned from
  deserialization); it never worked on the two fixed surfaces but did on a
  third, so all unit tests were green. **Only re-running the live probe that
  produced the finding caught it.** Rule: every fix must be closed with the
  probe that produced the finding — not with a test.
- **The fix phase's own verification run also produces false alarms, and at a
  high rate.** In this phase, 3 of 4 "FAILs" were my own assertion mistakes
  (searching for a plain string inside unicode-escaped JSON, `psql` printing a
  boolean as `f` instead of `false`, test data that didn't reach the
  threshold). Printing the raw output separated all three within minutes; had
  I paraphrased, all three could have been reported as findings.
- **Existing tests breaking after a fix is the cleanest red-green evidence.**
  Three fixes broke tests that had pinned the old behaviour (one carried the
  old expectation in the test NAME itself). "Updating them to the new
  contract" is not weakening the assertion — but the distinction needs to be
  written explicitly in the report, or the reader can't tell the two apart.
- **A subagent ran a destructive git command because the brief hadn't
  forbidden it** (`git checkout -- <file>`, discarding unstaged changes). No
  data was lost in this run — the state at the start of the run and the diff
  at that moment together proved it — but having to prove it is itself
  evidence the brief was incomplete.

**Additional proposals:**

- **P1 — a destructive-command ban must be added to executor/implementer
  subagent briefs.** `git checkout --`, `git reset`, `git stash`, `git clean`,
  file deletion: none may run without approval, and "reverting my own
  temporary change" is not an exception (it also reverts someone else's
  unsaved change in a shared file). One line in SKILL.md's subagent section.
- **P1 — the "close the fix with the finding's own probe" rule must enter
  Phase 5.** Phase 5 currently says "add a regression test + re-run"; there's
  no guarantee the test runs the *right* thing. The probe that produced the
  finding must run alongside the test, not in place of it.
- **P2 — the report template must distinguish "an existing test broken by the
  fix and updated to the new contract" from "a newly written test".** The
  former is red-green evidence and the most valuable line in the report;
  appearing in the same list as the latter makes that evidence invisible.


---

## 2026-09-07 · L3 · in-browser LLM client + web frontend (fix mode on)

**Caught thanks to structure:**

- **The category catalogue's "localisation/edge data" row** found a
  doc-language bug that never appeared in the feature list: because the doc
  language didn't follow the UI language, CSS `uppercase` applied the wrong
  language's uppercasing rules, and one of that language's most frequent
  letters printed corrupted. No functional scenario looks at this; it was
  looked at because the row exists.
- **The "read the basis from source-code intent comments too" approach**
  directly produced two S2s. In this codebase the comparison was possible
  because intent was written explicitly in comments: the gap between what the
  comment promised and what the code did would be invisible in an undocumented
  project.
- **"Suspect your own harness first on an unexpected FAIL cluster"** fired
  three times and was right all three (a masking regex hid its own control
  scope; an in-place-mutated array broke an assertion; a shared test helper
  was swallowing a prop). None of the three ever reached the report.

**Cost/noise:**

- **The level table doesn't account for the "execution burns money" case at
  all.** Here every scenario was a real paid model call. The case list was
  classified by execution cost (deterministic / interface / paid) and depth
  was shifted toward the cheap bucket — this was improvised, not a rule.
- **The "fix everything" mode in the same run silently traded off scope.** 11
  of 12 findings were closed, but 57 of the 122 designed cases never ran,
  including **all** of the concurrency, resilience, and exploratory charters.
  The level table says "60-120 cases run" for L3; this run gives the
  impression it kept that promise, but it didn't.

**Things I improvised that should be rules:**

- Diffing, at the source level, between the capabilities a dependency
  **declares** and the capabilities the client **consumes**. This run's most
  serious finding came from exactly this, and it was completely invisible to
  black-box testing: nothing errored, the system just silently misbehaved.
- For a changed default/factory function, search for **tests that mock that
  factory**. Those are exactly the tests that will stay green.

**Non-negotiable strained:**

- **#4 (test intent, not existing behaviour)** was strained in two places for
  two different reasons: (a) an existing test had deliberately pinned the
  wrong behaviour (its comment said so explicitly); (b) the written basis
  itself had been superseded by a later user decision, and the file didn't
  know it. The second is a case the rule never addressed at all.

**Proposals:**

- **P1 — a "the basis can be stale" rule, into `references/oracles.md` and
  Phase 0.** A written basis (plan, spec, design decision) can have been
  superseded by a **later** instruction in the project. If the basis is an
  artifact, check whether a newer decision supersedes it; if so, the newest
  decision governs and the finding is written **against the doc, not the
  code** ("the plan is stale at this point"). Otherwise correct code gets
  confidently reported as FAIL. SKILL.md Phase 0 already says "the basis
  itself can be defective", but only for *contradiction/gaps*; being
  *superseded later* is a different, sneakier case.
- **P1 — a new heuristic in `references/oracles.md`: "declared capability /
  consumed capability diff".** When a system talks to a dependency that
  declares its capabilities (protocol capabilities, schema hints,
  annotations, event types, pagination metadata, cache headers), compare at
  the source level which of the declared ones are actually **consumed** in
  code. An unconsumed declaration doesn't error; the system just silently
  misbehaves — and if the dependency's docs point the client toward that path,
  the consumer can't follow the instruction. This run produced an S2 this way.
- **P2 — an execution-cost dimension on the level table.** At kickoff, split
  cases into a cost class for scenarios whose execution burns money (paid
  API, real external call, long-running job), and shift depth toward the
  cheap class. The report should also state the cost distribution of the
  cases it ran — otherwise "I ran L3" means the same thing for two runs with
  wildly different costs.
- **P2 — an explicit warning for "L3 + fix mode".** When both are chosen for
  the same run, kickoff must **say** that scope will be traded off
  ("exploration first, or closing findings first?") or the run should be
  split in two. The current text never mentions this combination's effect on
  scope.
- **P3 — a "search for tests mocking that factory" line for a changed
  default/factory,** into Phase 0's existing-tests-reading item. This run
  found a real blind spot: 35 tests mocked the factory and injected the
  changed field, so nothing the change broke would have been visible.


---

## 2026-09-16 · L3 · backend API, multi-tenant permission boundary (report-only; the person running the QA = the person who wrote the change)

**Cases:** 138 · **Result:** 122 PASS / 3 FAIL / 13 BLOCKED · **Verdict:** GO WITH RISK
(0 findings attributable to the change; 2×S2 pre-existing and open) · **Skill version:** 1.7.0

**Caught thanks to structure:**

- **Category 6 (combinations)** set up as a decision table produced a "surface ×
  injection point × tenant relationship" matrix and carried the sweep from
  2-3 endpoints to 14 surfaces. An improvised run stops at the first three of
  these.
- **Non-negotiable #8 (tenant isolation is a standing assertion, not a
  category)** changed the shape of the claim: from "did it return 403" to
  "which tenant did the record land in". This reframing directly produced the
  run's heaviest finding (a payment could attach to another tenant's
  sub-record) — and along the way showed that a claim expecting 403 would keep
  passing even if the fix were reverted.
- **Category 4 + the consistency oracle** caught sibling endpoints behaving
  differently at the same boundary value (one 400, the other 500). Not
  findable by exploration.
- **The "read the basis backwards" rule** showed a deleted field wasn't
  covered by any case; closed within the same run.

**Cost/noise:**

- An executor agent shied away from making a real external-provider call and
  left the case as NOT RUN; the provider was actually reachable and safe, and
  the case completed in two calls. The environment manifest records whether a
  dependency *exists*, not whether *calling it is safe*.
- Two fake FAILs, both from the same cause: an enum's identifier name was used
  as if it were the wire value. The existing "a synthetic value must be
  valid" rule talks about format checking; this mistake passes format
  checking because the shape is correct.
- 13 of 138 cases were BLOCKED, almost all from missing **fixture** data. The
  environment inventory covered dependencies, not fixtures; all 13 were
  discovered mid-run.

**Things I improvised that should be rules:**

- **A/B differential run.** Stand up the pre-change build side by side against
  the same data store and run every finding against both. The run's single
  most valuable move: "all eight findings are pre-existing" became a
  measurement instead of a claim, surfaces that must not be touched could be
  compared byte for byte, and the fix's red-green came out without writing a
  separate test.
- **Proving the baseline's validity first.** No A/B result was trusted until
  it was shown that the old process's start time was before the first edit.
  Without this step, the technique can prove the opposite of its own result.

**Non-negotiable strained:**

- **#6 (self-certification ban)** was strained from a place the rule doesn't
  name: the person who wrote the change also designed the case list. The rule
  wants execution independence, not design independence. The boundary-value
  inconsistency in the new code they themselves wrote was found not by the
  person who designed the list, but by an independent agent comparing the
  same endpoints against a sibling endpoint.
- **The verdict rule collided with reality:** "an open S2 on a critical flow →
  NO-GO" was blocking a fix that closed a proven S1, because of a pre-existing
  S2. The rule doesn't ask whose finding it is.

**Proposals:**

- P1 — differential (A/B) execution technique + baseline validity proof → APPLIED (v1.8.0)
- P1 — an attribution dimension on verdict rules → APPLIED (v1.8.0)
- P2 — Phase 0 fixture inventory → APPLIED (v1.8.0)
- P2 — design independence added to non-negotiable #6 → APPLIED (v1.8.0)
- P3 — "read the value, not the name" added to the synthetic-value rule → APPLIED (v1.8.0)
- **P3 — OPEN:** a "safe to call" column on the environment manifest's
  dependency table. It currently only says *real / mock / none*; an executor
  agent that doesn't know whether an external call has an irreversible cost
  plays it safe and skips the case. If it recurs in a future run, it escalates
  to P2.

---

## 2026-09-22 · L3 · backend API / service layer (concurrency + lock/isolation fix; report-only, then 3 fixes)

**Cases:** 41 · **Result:** 33 PASS / 2 FAIL / 6 BLOCKED · **Verdict:** GO
(0 findings attributable to the change; 2 findings measured pre-existing) ·
**Skill version:** 1.8.0

**Caught thanks to structure:**

- **A/B differential run (§11), this time for validity rather than
  attribution.** Running the pre-change build against the same data store
  proved, in a single measurement, both that the bug was real and that the
  harness could see it: the old code distributed four times the cap without
  ever issuing a rejection, the new code stopped exactly at cap. Without this
  measurement, the new build's green result and "concurrency simply never
  collided" would have been indistinguishable.
- **The requirement to run category 9 against a real dependency.** The
  project's existing suite was fully green and carried zero information about
  the change's central claim — a mocked lock sees neither a real Prisma lock
  nor the isolation level. Every real finding in this run came from the
  hand-built real-service/real-database harness.
- **Separating a finding by whether it belongs to this change** showed that
  both timing-sensitivity findings were pre-existing; one of them (the half
  tied to the cap) was still charged to this change anyway, because the
  number the fix clipped ran through exactly that path. Reporting the two as
  a single finding would have put pressure to fix the out-of-scope half too.
- **Arguing against your own verdict once (#7)** forced measuring an
  independent design agent's unverified hypothesis (that refund-path counters
  could go negative) before it entered the report; the measurement refuted
  the hypothesis and the finding was dropped.

**Cost/noise:**

- **A timing finding was overstated tenfold on first measurement.** 13 of 25
  rounds had been counted as "bad"; in most of those rounds the condition had
  actually kicked in after everyone was already done — expected behaviour.
  Once benign orderings were filtered out, the real number was 3 in 30. Had
  it entered the report, fix priority would have been miscalculated.
- The report and suite files were written to the main working copy instead of
  the worktree; noticed when an `Edit` said "file not found". Verifying the
  absolute path once would have cut this off from the start.

**Things I improvised that should be rules:**

- **Test at the level the claim lives at.** A concurrency/lock/
  isolation/atomicity claim isn't considered covered by a mocked suite no
  matter how green it is; at least one case must run against the real
  dependency, or the report must say so in those words.
- **Red-green isn't a formality for concurrency.** A green result carries no
  information until the same harness is shown to go red on unfixed code.

**Non-negotiable strained:**

- **#3 (red-green)** fell short specifically for concurrency: the rule says
  "the test must fail before the fix", but a concurrent scenario simply never
  occurring produces the same output. The rule needed to name this case
  specifically.

**Proposals:**

- P1 — red-green requirement for concurrency (extension of #3) → APPLIED (v1.9.0)
- P1 — new non-negotiable #9: the test must run at the claim's own level → APPLIED (v1.9.0)
- P2 — filtering benign orderings before counting a timing finding
  (Phase 3) → APPLIED (v1.9.0)
- **P3 — OPEN (carried forward):** a "safe to call" column on the environment
  manifest's dependency table (from the 2026-09-16 run). Didn't recur in this
  run — no case needed an external provider call — so it stays open as P3.

## 2026-10-01 · L3 · backend API, permission restriction on a state-mutating endpoint (fix mode on)

**What the structure caught:**

- **The A/B differential decided the verdict.** Every S2/S3 found reproduced
  identically on the pre-change build, so attribution was measured, not argued,
  and the change itself was proven red-green: the behaviours it removed still
  succeeded on the baseline and were rejected on the branch.
- **Design independence paid off.** The case list from a subagent that saw only
  the diff and the basis predicted two defects from code reading (a lock-order
  deadlock, sub-unit amounts desyncing two columns of different precision);
  both reproduced against the real dependency.
- **A conservation oracle** (the sum of a conserved quantity before vs after) turned "two stores of
  the same quantity disagree" from a suspicion into a number.

**Cost/noise:**

- The documented credential for the auth surface no longer worked. Recovering
  cost time; the fix was a QA-only bootstrap that skipped only the signature
  check and kept every other layer real, declared in the report.
- Two expectations were too strict on the error *code* while the basis only
  fixed the *outcome* (rejected, no side effect); reclassified in Phase 3.

**Things I improvised that should be rules:**

- **Lead execution by script when fixtures cannot be isolated** (see v1.10.0).
- **When the documented access method breaks, a harness that disables only the
  credential check — and nothing else — is acceptable**, if the report names
  exactly which layer was bypassed and lists real-credential validation as
  BLOCKED.

**Non-negotiable strained:** none.

**Proposals:**

- P2 — lead may execute when fixtures cannot be isolated, scripted, declared → APPLIED (v1.10.0)
- P3 — document the "bypass only the credential check" fallback → APPLIED (v1.11.0, recurred on 2026-10-02)
- **P3 — OPEN (carried forward):** "safe to call" column on the environment
  manifest's dependency table. Did not recur.

## 2026-10-02 · L2 · backend API, idempotency keys and asynchronous batch verdicts (fix mode on)

**What the structure caught:**

- **The cold design review found where the defects were.** A reviewer that saw
  only the diff, the basis and the finished list added nine cases; both confirmed
  defects came from them (a replay answered by a later rule instead of the
  idempotency rule, and a write-off that ignored a durable ownership link).
- **A/B made every ticket a measured red-green**, including a real race: the
  baseline went 0/10 on the same harness that went 10/10 on the branch, with
  start timestamps proving the overlap.
- **Exercising the real scheduler** (backdating a row and waiting for the tick)
  verified a sweep fix through its actual trigger instead of a direct call.

**Cost/noise:**

- Running two builds side by side would have mixed their async jobs (shared
  queues); A/B had to be sequential.
- Three false alarms, all harness-made: an org-wide aggregate moved by parallel
  executors, a downstream record read before its async writer ran, and an
  assertion on a field that lived one level deeper. All caught in Phase 3.

**Things I improvised that should be rules:**

- Stub only the lookup in front of the logic, with failure-mode switches
  (→ v1.11.0).
- Sequential A/B for async probes (→ v1.11.0).

**Non-negotiable strained:** #6 — the lead wrote the code; design independence
came only from the reviewer subagent, which is why it is now the default.

**Proposals:**

- P2 — cold case-list review default at L2+ → APPLIED (v1.11.0)
- P2 — sequential A/B for async probes → APPLIED (v1.11.0)
- P3 — lookup-dependency stub instead of blanket BLOCKED → APPLIED (v1.11.0)
- **P3 — OPEN:** a harness note in `test-data.md` — with parallel executors,
  assert through records attributable to the case, never through a shared
  aggregate, and wait for the downstream effect before counting it.

## 2026-10-05 · L3 · backend API, per-entity attribute written on the load path (fix mode on)

**What the structure caught:**

- **A/B found a regression no response showed.** A new write path moved a
  persistence-stamped "updated at" column that reports sort and filter on. Only
  reading the stored row back on both builds revealed it.
- **A real-database race harness with a deliberate mutant** (last writer wins)
  proved it could go red before its green was trusted, and found a lost-update
  defect the doubles-based suite could not see.
- **Mutation testing on the new pure modules** left three survivors, all at
  exact-instant boundaries (`<` vs `<=`); the tests added for them brought the
  score to 100 %.
- **The cold reviewer** added 22 cases and corrected 14 expectations.

**Cost/noise:**

- Two false FAILs from expectations copied from REST convention (a dedicated
  not-found status) in a project whose error layer answers every business error
  with one generic client-error status.
- A long-running tool started from a foreground background-subshell died with
  the call; it had to be re-run as a tracked background task.

**Things I improvised that should be rules:**

- Read persistence-maintained columns back on both A/B builds (→ v1.12.0).
- Run a hand-written query binding a time value under several DB session time
  zones (→ v1.12.0).

**Non-negotiable strained:** #6 — the lead wrote the change; design independence
again came from the cold reviewer subagent only.

**Proposals:**

- P2 — persistence-maintained columns in the A/B diff → APPLIED (v1.12.0)
- P2 — project conventions outrank textbook expectations → APPLIED (v1.12.0)
- P3 — session time zone sweep for hand-written time queries → APPLIED (v1.12.0)
- The 2026-10-02 P3 (assert through case-attributable records, never a shared
  aggregate) stays OPEN; this run had no recurrence.

## 2026-10-05 · L2 · backend API, alternative input for a scheduling rule, fix mode

**Cases:** 72 (15 from the cold review, 5 discovered during fix verification) ·
**Result:** 0 FAIL after fixes; 3 S3 found (2 fixed red → green, 1 deferred by the
user) · **Verdict:** GO WITH RISK.

**What the structure caught:**

- The cold case-list review produced both fixed defects:
  - an accepted value under which the rule never applied;
  - a missing upper bound that a sibling input already had.
- The executors passed every case in the original list. The defects were outside
  what the mind that wrote the change had thought to test (non-negotiable #6
  working as designed).
- The lock-stripping mutant turned a green race harness into evidence: real 0/20,
  mutant 15/20.

**Cost/noise:**

- One false FAIL. The executor computed a money delta for a partial operation
  itself, and got it wrong.
- One harness FAIL caused by module-resolution flags when importing application
  code into a script. That is project-specific, so it was recorded in the
  project's environment manifest.

**Things I improvised that should be rules:**

- Checking each accepted extreme against the instant at which the feature acts
  (→ the Effect oracle).

**Non-negotiable strained:** #6 again. The lead wrote the change, and design
independence came only from the cold reviewer.

**Proposals:**

- P2 — the Effect oracle, pointed at by the cold review → APPLIED (v1.13.0)
- P3 — expected deltas written as numbers → APPLIED (v1.13.0)
- The 2026-10-02 P3 (assert through case-attributable records, never a shared
  aggregate) stays OPEN; this run had no recurrence.
