# Release gate: verdict rules + pre-production checklist

Read this before writing the verdict. The point of a verdict is to convert a long
report into a decision the developer can act on in ten seconds.

## Verdict rules

Apply mechanically, then state the rule you applied:

| Verdict | Condition |
|---------|-----------|
| **NO-GO** | Any open **S1**; or any open **S2** on a critical flow (`.qa/critical-flows.md`); or a data-migration/rollback risk that can't be undone; or a critical-flow smoke test that failed; or coverage so blocked that the change is effectively untested (a mandatory catalogue row at zero because the environment prevented it) |
| **GO WITH RISK** | No open S1; open S2 outside critical flows, or S3s that matter, or a mandatory category deliberately skipped — each listed with its risk and the reason it's acceptable now |
| **GO** | No open S1/S2; remaining findings are S3/S4 with known workarounds; every mandatory catalogue row has at least one executed case; critical-flow smoke green |

Write it as: `NO-GO — 1 open S1 (coupon can be applied twice, #3)`.

### Attribution — the change is judged on what it caused

The table above says "open S1/S2" without asking *whose*. Applied literally, a
change that fixes a serious defect gets blocked by an unrelated defect it merely
walked past — and blocking it leaves **both** in production. So the verdict is
computed on the findings **attributable to the change**, and attribution is a
measurement, not a courtesy (`references/techniques.md` §11 for how to measure
it; without a baseline, the label is provisional and so is this rule's benefit).

- A finding **verified as pre-existing** — reproduces identically on the
  pre-change build — does not push the verdict to `NO-GO`. It caps it at
  `GO WITH RISK`, is reported at its true severity, and leaves the run as its own
  ticket with an owner. Never silently downgrade it to a footnote because it
  isn't "yours".
- **Two exceptions put it back on the change's account:** the change was supposed
  to fix it (then it's a failed fix, full stop), or the change makes it more
  reachable, more severe, or newly exploitable (then it's a regression, whatever
  its age).
- A finding that appears **only on the new build** is a regression and carries
  full weight — an open S1 there is `NO-GO` however small the diff.
- Say the attribution split in the verdict line, because it's the part a reader
  will otherwise get wrong: `GO WITH RISK — 0 findings attributable to the
  change; 2×S2 pre-existing and open (verified via A/B), separate ticket`.

The honest framing for the user, when a pre-existing finding is what's holding
the verdict below `GO`: shipping does not make it worse, and not shipping does
not make it better — but the two decisions are still theirs, so give them the
comparison rather than a single word.

**Level caps the verdict.** An L1 (smoke) run cannot produce a `GO` — most of the
ground was never tested, so the best it can say is `GO WITH RISK` with the
untested categories listed. `GO` is available at L2, and only an L3 run with the
pre-production checklist walked can say `GO` for a change that touches money,
data migration, permissions or a public contract.

Guardrails on your own verdict:

- Never upgrade to `GO` because the user seems to be in a hurry, and never
  downgrade to look thorough. The rule decides, not the mood.
- `BLOCKED`/`NOT RUN` cases are not `PASS`. If they cover a mandatory category,
  the verdict is at best `GO WITH RISK` — and say what's untested.
- If Phase 5 fixes happened, re-issue the verdict *after* re-running; a verdict
  from before the fixes is stale.
- Ambiguous requirements go to open questions and, if they affect a critical
  flow, they hold the verdict at `GO WITH RISK` until the user decides.

## Pre-production checklist

Beyond the feature's own behaviour, these are what break releases. Walk them for
any change that is about to ship; mark each `OK` / `RISK` / `N/A`.

**Migration & data**

- Migration runs forward on a copy with realistic volume, in acceptable time
  (does it lock a big table? on MySQL/Postgres at size, prefer an online
  schema-change tool such as gh-ost or pt-online-schema-change).
- Existing rows get sensible defaults; new NOT NULL columns don't fail on them.
- Rollback path exists and was tried — or, if one-way, that is stated explicitly.
- Backfill is idempotent and re-runnable after a partial failure.
- Indexes exist for every new filter/sort/join the feature introduced.

**Expand/contract — the pattern that makes rollback possible.** A schema change
that renames or drops in one step makes the release irreversible: old code can no
longer run against the new schema. Require the change to be split, and check
which step this release is at:

1. Add the new structure, additive and nullable. *(reversible)*
2. Write to both old and new; keep reading from old. *(reversible)*
3. Backfill existing rows. *(reversible)*
4. Verify the new path — shadow reads, counts and checksums against the old.
5. Switch reads to the new structure. *(first behaviour change; still reversible)*
6. Stop writing to the old structure. *(reversible)*
7. Drop the old structure. **Point of no return.**

**Before an irreversible step (7, or any delete/drop/non-additive change):** a
restorable backup exists (and restore was actually tried, not assumed); the new
path has run a full deploy cycle without a rollback; nothing still references the
old structure — including background jobs, reports, exports and analytics; and
the user has explicitly approved it. If any of those is missing, the verdict is
`NO-GO` on the drop, regardless of how healthy the feature looks.

**Backward compatibility**

- Old clients still work: nothing removed or renamed in a response, no newly
  required request field, no narrowed type or enum.
- Clients you can't force-update (mobile apps, third-party integrations, cached
  SDKs) keep working on the old contract.
- Queue/event consumers tolerate the new message shape, and old in-flight
  messages still process.
- Two versions running side by side during deploy (rolling release) don't corrupt
  shared state.

**Configuration & flags**

- Every new env var/secret is documented and present in the target environment;
  behaviour on a missing or malformed value is a clear startup failure, not a
  silent default.
- The feature works with its flag **off** as well as on, and flipping it at
  runtime doesn't leave records half-processed.
- No hardcoded localhost/dev host, test key, or debug flag left in the diff.

**Security & privacy**

- New endpoints have auth and authorisation, including the less-used verbs and
  bulk/export routes.
- No secret, token, API key, card number or PII in logs, error bodies, URLs or
  analytics events.
- Rate limits apply to anything expensive or enumerable that the change added.
- Dependencies added in the diff are known-good (no typo-squat, no abandoned
  package) and don't pull in a known CVE.

**Operability**

- Failures are logged with enough context to diagnose without a repro.
- Something observable tells you the feature is healthy in prod (metric, log
  line, dashboard) — otherwise the first sign of breakage will be a user.
- A plan exists for reverting the change itself (revert commit, flag off, or
  documented manual step).

**Housekeeping**

- Test data created during the run is cleaned up, or listed if left behind.
- New regression tests are committed and run in CI, not just executed once here.

## After the deploy — the checks that belong to the release, not the branch

The gate doesn't end when the code merges. Two cheap habits catch most of what
slips past pre-release testing, and both are safe to run against production
because they don't mutate customer data:

**Read-only prod smoke, within minutes of deploy.** A handful of GETs plus one
auth round-trip against a dedicated, clearly-marked internal test tenant: health
and readiness endpoints, login, the critical flows' read paths, one report/export
that exercises real queries. Strictly read-only or idempotent, never a write on
customer data, never a destructive scenario — this is the one prod interaction
the "no production testing" rule permits, and only when the user asked for it.
Failure means roll back, not investigate later.

**New-error-signature diff.** Take the error fingerprints (exception type + top
frame + endpoint) from the window before the deploy and the window after. Any
signature that appears only in the "after" window is a same-day investigation:
new signatures correlate with the change far better than error-rate averages,
which stay flat while a small cohort of users breaks. If the project has an SLO
or error budget, check the burn rate too — a release that eats a large share of
the budget deserves a postmortem, not a shrug.

Also worth confirming once, right after the release: the feature flag can still
be flipped off cleanly, and the alert or dashboard that would tell you this
feature is unhealthy actually exists. A feature nobody can observe will report
its first failure through a customer.

## What a human still needs to check

Include this honestly in every report — the shortlist changes per feature, but
these recur:

- Real payment rails, real bank/3D-secure flows, real e-mail/SMS deliverability.
- Third-party behaviour that the sandbox doesn't reproduce.
- Visual polish and UX judgement: does it *feel* right, is the wording right for
  users, does the layout look right on a real device.
- True load and long-running behaviour (hours of traffic, memory growth, cron
  interactions).
- Anything requiring credentials, devices or accounts you don't have.
