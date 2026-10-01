# Reporting: severity rubric, template, quality bar

## Severity rubric

| Severity | Meaning | Examples |
|---|---|---|
| **S1 – Critical** | Data loss/corruption, security or tenancy breach, money wrong, feature unusable for everyone, no workaround | Another tenant's records readable; payment charged twice; stock goes negative; totals wrong; auth bypass; unrecoverable state |
| **S2 – High** | Core scenario broken or silently wrong for a real subset of users; ugly workaround only | Validation missing so bad data persists; illegal state transition accepted; list leaks soft-deleted rows; race creates duplicates; 500 on a normal flow |
| **S3 – Medium** | Edge case wrong, poor error handling, misleading message, wrong-but-recoverable behaviour | Unhelpful/raw error text; boundary off-by-one on a rare limit; pagination duplicates a row; timezone shifts a display date |
| **S4 – Low** | Cosmetic, wording, minor UX, inconsistency with no functional impact | Label truncation; inconsistent date format; missing loading indicator |
| **Risk** | Not reproduced (yet) but code/behaviour suggests a real problem | Unbounded query with no index; `catch {}` swallowing errors; missing idempotency key |
| **Question** | Requirement is ambiguous — behaviour may be intended | "Should an expired coupon on a pending order stay valid?" |

Silent-and-wrong beats loud-and-broken in severity: a wrong number nobody
notices is worse than an error page.

## Report template

Use this structure. Keep it scannable — the worst finding goes at the top.

```markdown
# Test Report — <feature> — <date>

## VERDICT: <GO | GO WITH RISK | NO-GO>
> Rationale: <the rule that produced it, e.g. "1 open S1 (#3)">
> <If NO-GO: what must be fixed before this can ship.>

## Summary
- Level: <L1 smoke | L2 standard | L3 release gate | focused> — <user-selected / default>
- Scope: <branch/commit, environment, tier plan (A: … / B: … / C: … / D: …)>
- Result: <N> scenarios run — <N> PASS / <N> FAIL / <N> BLOCKED / <N> NOT RUN
- Findings: <N>×S1, <N>×S2, <N>×S3, <N>×S4, <N> risk, <N> open question(s)
- Most critical finding: <one sentence>

## Findings (by severity)

### [S1] <short title>  ·  `KPN-021`
- **Where:** <endpoint / screen / file:line>
- **Reproduction:**
  1. <exact step, with the exact request/input>
  2. ...
- **Expected:** <...> — *basis:* <requirement / schema / constraint / doc>
- **Actual:** <...>
- **Evidence:** <real response body / log line / DB row / screenshot path>
- **Verification:** reproduced <N> times from a clean state; alternative
  explanations ruled out: <stale build / bad test data / malformed request / intended behaviour>
- **Likely cause:** <file:line + one-line diagnosis, if known>
- **Impact:** <who is affected and how badly>
- **New:** <introduced by this change | pre-existing>

### [S2] ...

## Risks and observations
- <not reproduced, but worth attention — with reasoning>

## Open questions
- <ambiguous requirement — needs a product decision>

## Coverage summary (by category)

Every row of the catalogue must appear here; a zero row needs its reason stated.

| # | Category | Scenario | PASS | FAIL | NOT RUN | Note / rationale |
|---|----------|---------|------|------|---------|----------------|
| 1 | Happy path | 5 | 5 | 0 | 0 | |
| 2 | Functional mechanics | 8 | 7 | 1 | 0 | |
| 3 | Negative / validation | 11 | 9 | 2 | 0 | |
| 4 | Boundary value | 9 | 8 | 1 | 0 | |
| 5 | Equivalence class | ... | | | | |
| 6 | Combination (decision table / pairwise) | ... | | | | |
| 7 | Permission / tenancy | ... | | | | |
| 8 | State & sequence | ... | | | | |
| 9 | Concurrency & idempotency | ... | | | | |
| 10 | Data integrity | ... | | | | |
| 11 | Edge data | ... | | | | |
| 12 | Error handling & resilience | ... | | | | |
| 13 | Security | ... | | | | |
| 14 | Regression & integration | ... | | | | |
| 15 | Exploratory / bug hunting | ... | | | | |
| 16 | Performance | 0 | | | | dev DB has a single record — no meaningful measurement |
| 17 | Compatibility / responsive / a11y | ... | | | | |
| 18 | Localisation & formats | ... | | | | |
| 19 | Config / migration / deploy | ... | | | | |
| 20 | Observability | ... | | | | |

## Case list

Full list in the file: `.qa/suites/<feature>.md` (or `test-cases-...md`).
In the report, either link to the file or copy the table here.

| ID | Tier | Category | Scenario | Expected | Actual | Status |
|----|------|----------|---------|----------|-------------|-------|
| KPN-001 | A | Happy | ... | ... | ... | PASS |
| KPN-014 | A | Boundary | ... | ... | ... | FAIL (S2) |
| KPN-041 | C | Critical flow smoke | ... | ... | ... | PASS |

## Excluded from scope
- <categories skipped due to level — which ones, why>
- <deliberately deprioritised — be honest>

## Automated tests added
- `path/to/test_file.py::test_name` — <what it guards> (was failing before fix X)

## Pre-production checklist
| Area | Status | Note |
|------|-------|-----|
| Migration & data (reversible? indexes?) | OK / RISK / N/A | |
| Backward compatibility (old clients, queue messages) | | |
| Config & feature flag (including the off path) | | |
| Security & privacy (auth, secrets/PII in logs, rate limit) | | |
| Operability (logging, metrics, rollback plan) | | |
| Housekeeping (test data, tests added to CI) | | |

## What a human still needs to check
- <real payment / e-mail deliverability / visual UX / real load / an account I can't reach>

## Suggested fix order
1. [S1] <finding> — <one-line fix idea>
2. [S2] ...

> In report-only mode the report ends here: "Would you like me to make these fixes?"

## Fixes made *(only if a fix was requested)*
| # | Finding | Fix | File | Verification |
|---|-------|----------|-------|-----------|
```

## Quality bar for a finding

A finding is only useful if someone else can act on it. Before writing one down:

- **Verified:** it survived the Phase 3 refutation pass — reproduced from a clean
  state, alternative explanations ruled out, checked against
  `.qa/accepted-behaviours.md`. An unverified item is a Risk, not a bug. When
  there is no human tester downstream, a false alarm is more expensive than a
  missed S4: it trains the reader to skim.
- **Reproducible:** exact input, exact steps, minimal case. "Sometimes fails" is
  not a bug report — find the trigger, or file it as a Risk and say what you saw.
- **Evidence, not adjectives:** paste the actual response, log line or DB row.
  Never invent output you didn't observe.
- **One bug per entry:** don't bundle three problems into one item; don't split
  one root cause into eight items (group them and name the root cause once).
- **Expected clearly justified:** cite the requirement, doc, schema constraint
  or convention that makes the actual behaviour wrong. If nothing justifies it,
  it belongs in Open questions, not Findings.
- **No blame, no drama:** describe behaviour, not the developer.

## Honesty rules

- Verdicts reflect what you actually ran. `NOT RUN` is respectable; a fake
  `PASS` destroys the value of the whole report.
- If the environment blocked you, say which cases were blocked and why.
- If you found nothing, the report is still the deliverable: the case list with
  its statuses shows what was attacked, which is what makes "no bugs found"
  believable.
- State the depth you chose and what you skipped. Untested-and-declared is fine;
  untested-and-implied-tested is a lie the user will discover in production.
