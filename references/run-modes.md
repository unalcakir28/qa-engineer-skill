# Run modes and parallel execution

Variants of the default process, not separate processes — every non-negotiable,
every phase and every evidence rule in `SKILL.md` still applies.

## PR mode

The user points at a pull request (e.g. "test this PR").

- Tier A is the PR's diff; run the normal phases at the chosen level.
- In addition to the standard report, produce a condensed PR-comment version:
  verdict, top findings with case IDs, one-line coverage summary, link to the
  full report.
- Post it to the PR **only with the user's explicit approval** — a review comment
  is outward-facing.

## Sentinel mode

The skill is wired to a scheduler (cron, CI) and nobody is there to answer.

- Unattended defaults apply: **L2, report-only**, stated at the top of the report.
- Scope: tier C critical-flow smoke + tier D rotation, plus tier A for whatever
  changed since the last logged run.
- Write the report and update the QA memory as usual; lead with any S1/S2 so it
  is the first thing a human sees.
- Leave retrospective proposals as `PROPOSAL — awaiting approval`.
- Never fix code, never prune data, never post anywhere external while unattended.

## Fix-verification mode

Issues were filed earlier (by you or by someone else), a developer reports them
fixed, and the user asks you to confirm. The question per issue is "does the
system now do what the issue said it should", not "does the change do what the
developer said it does".

- **Basis per issue:** the issue's own expected behaviour and its repro. The
  developer's comment tells you where to look, but it is not the expectation. A
  fix that removes the symptom but covers a narrower case than the issue
  describes is PARTIAL.
- **Order:** by severity, highest first, so the riskiest answers arrive first.
- **Per issue, at least:**
  - re-run the original repro on the deployed revision (read the revision
    first);
  - test both sides of any new limit or rule, with multi-byte or otherwise
    awkward input where the unit could be ambiguous;
  - re-run the equivalence classes that the original finding belonged to, not
    only the sample that was reported;
  - check the data store for side effects;
  - for a concurrency or idempotency claim, use the red recorded in the earlier
    run, or establish one, before trusting a green.
- **Verdicts:** FIXED, PARTIAL, BLOCKED (with what would unblock it) or NOT RUN.
  A PARTIAL comment has two parts: what is fixed, and what is still open with a
  minimal repro.
- **Update the tracker issue by issue**, as each verdict lands, not in a batch
  at the end. Post in the tracker's language (see `SKILL.md`, Language).
- **User-gated steps:** some steps only the user can do: a real payment, a
  consent tick, an action on an account only they control. List them at design
  time and batch them into one joint session. Give numbered instructions, and
  check the data store after each step the user reports as done.
- **Unexpected events near the action under test:** trace them by entity id to
  their source before you draw any conclusion. A scheduled job running at the
  same minute is not the change under test.
- **Close with a short regression** over the areas the fixes touched, so a fix
  that broke a neighbour is caught in the same pass.

## Parallelising a large list

A 60-case list executed in one context degrades near the end — attention drifts
to wrapping up. At **L3** (and any L2 run that grew past ~40 cases), split
execution by **category group** across Sonnet subagents, one agent each,
reporting back as structured statuses and evidence:

1. auth, permissions, tenancy (7)
2. validation, boundaries, equivalence, combinations (3–6)
3. state, sequence, concurrency, idempotency (8–9)
4. data integrity, migration, observability (10, 19, 20)
5. resilience, error handling, security probes (12, 13)
6. functional mechanics, happy path, UI/localisation (1, 2, 17, 18)
7. regression, integration, critical-flow smoke (14, tier C)

Give each agent the case-list rows it owns, the environment facts it needs, and
the evidence rule (real command, real output, status per row). Merge the results
yourself, run Phase 3 verification on the strong model, and keep ownership of the
verdict — you are the one signing the report.

**When the lead may execute instead.** Delegation exists to protect the lead's
context and to move token spend to the cheaper model; it is not a goal of its
own. If the groups cannot be given isolated fixtures — the cases share one
seeded graph of records that cannot be cloned cheaply, or they deliberately
drain or contend on the same finite resource — parallel agents would poison each
other's assertions. In that case the lead may run the list itself, provided the
cases are mechanised as scripts (each case a function that records status and
raw evidence) rather than walked by hand, so context does not degrade towards
the end of the list. Say so in the report and in the metrics row, with the reason
("execution by the lead: shared fixture could not be isolated per group").

**Lanes for single-instance resources — planned at design time.** Some
resources cannot be cloned per agent: a session a human signed into by hand, the
one physical device, an account only a person can create. Every case that needs
one runs in that resource's **lane**, and each lane has exactly one executor at a
time.

- List these resources in the environment manifest.
- Give every case row a lane tag in Phase 1, not at dispatch.
- Before dispatching, count the rows that no executor owns. A row without an
  owner is not run by anyone, and it looks exactly like a row someone forgot to
  report. Each such row is run by the lead or marked `NOT RUN` with the reason.
- Every executor brief forbids actions that end or degrade a shared session or
  account: logging out, revoking a token, changing its role, banning or deleting
  it. Cases that need such an action are scheduled last in their lane, and the
  lead runs them.
- A lane is identified by the signed-in identity, never by the tooling's handle
  for it. Handles can be renamed or reassigned when the tooling reconnects. The
  brief gives the expected identity, the executor reads the identity before
  every write, and the executor stops if it does not match. Executors never
  select, switch or re-attach a lane's session.

Seven rules that keep a parallel run honest:

- **Each agent seeds its own isolated fixture set** (own org/tenant/user/records,
  prefixed identifiers). Parallel agents must never share mutable test data — one
  agent's state-transition case silently breaks another agent's assertion, and
  the resulting FAILs look exactly like real bugs. If a group *must* mutate
  shared or global state (DDL, config, a shared counter), run that group alone,
  not in parallel.
- **A mutant touches only its executor's own fixtures.** A deliberately broken
  copy of a guard runs inside the executor's own process, but what it writes lands
  in the shared store. If the mutated component scans or schedules over shared
  state — a periodic sweep, a poller, a queue consumer — scope the mutant to the
  executor's own record ids, or run it when no other executor is active, or
  against an isolated store. Otherwise it rewrites other agents' records, and
  every result that touched them has to be re-checked against a time window.
- **An executor that waits on real time records as it goes.** Cases that wait for
  scheduler ticks or timeouts make an agent run long, and a long agent can stall.
  Each case's status and evidence are written the moment the case finishes, so a
  stalled executor can be resumed for its summary without re-running anything.
- **A brief cannot grant what the lead does not hold.** Some actions need the
  user's approval every time: accepting terms or consent, paying, sending
  messages to real people, publishing. An executor brief never permits them,
  even when a case seems to need them. Those cases go back to the lead, which
  asks the user or marks them `BLOCKED (user-gated)`.
- **Anything an agent posts is idempotent.** An agent that writes to an external
  tracker or log checks first whether the same entry already exists. A retried
  or resumed agent otherwise posts duplicates. New local ids (finding numbers)
  are read from the file at the moment of writing, never taken from the brief.
- **Raw evidence is part of the contract.** Require each agent to return, per
  case, the actual request and response (or command and output) — not a
  paraphrase. Without the raw exchange you cannot tell a real FAIL from the
  agent's own malformed request, or a real PASS from a request that hit the
  wrong endpoint and got a cheerful 200.
- **A subagent's word is a claim, not a result — in both directions.** Every FAIL
  gets re-run by you before it becomes a finding (Phase 3). And a wrong PASS is
  just as possible as a wrong FAIL: sample-verify a few PASS rows from the
  highest-risk categories (money, permissions, state, concurrency) by re-running
  them yourself. If a sampled PASS doesn't hold, re-verify that agent's entire
  group.
