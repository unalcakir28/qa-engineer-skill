# The case list: format, IDs, statuses, lifecycle

The case list is the artefact that makes a run auditable instead of a story about
a run. Write it before execution, keep it updated during execution, and hand it
over with the report.

## Where it lives

- With QA memory: `.qa/suites/<feature>.md` — reused and extended on later runs,
  so it grows into the project's case inventory.
- Without `.qa/`: `test-cases-<feature>-<YYYY-MM-DD>.md` next to the report.

One file per feature/area, not per run. Runs are recorded inside it (see
`## Run history` below) and in `.qa/regression-log.md`.

## Format

```markdown
# Test Case List — Coupon discount
Case ID prefix: **KPN** (unique project-wide; cross-references use `KPN-xxx`)
Branch/commit: feature/coupon @ abc1234 | Environment: local | Author: qa-engineer
Tier plan: A=coupon discount · B=order total, stock · C=login+order smoke · D=webhook

## Summary
Total 47 cases — 5 happy, 8 functional, 11 negative, 9 boundary, 4 permission,
6 state/concurrency, 4 data integrity. Skipped: #17 a11y (UI unchanged),
#16 performance (dev DB has a single record).

## Cases

| ID | Tier | Category | Scenario | Precondition / data | Steps | Expected | Status | Evidence |
|----|------|----------|---------|-----------------|---------|----------|-------|-------|
| KPN-001 | A | Happy | Valid coupon applies 10% discount | active coupon `SAVE10`, cart ₺100 | POST /orders (coupon=SAVE10) | 201, total=₺90, discount=10 in DB | PASS | resp 201, order#881 |
| KPN-014 | A | Boundary | Coupon amount equals cart total | coupon ₺100, cart ₺100 | POST /orders | 201, total=0, never goes negative | FAIL (S2) | total=-0.01 → BUG-016 |
| KPN-021 | A | Concurrency | Same coupon on 20 parallel requests | single-use coupon | 20× POST in parallel | 1 succeeds, 19 rejected | FAIL (S1) | 3 succeeded → BUG-017 |
| KPN-033 | B | Data integrity | Total stays consistent after discount cancellation | KPN-001's order | DELETE /orders/881 | stock restored, discount record deleted | PASS | verified in DB |
| KPN-041 | C | Critical flow | Login smoke | — | POST /auth/login | 200 + token | PASS | |
| KPN-048* | A | Exploratory | Coupon expires mid-request | coupon expires in 5s | POST /orders (t=6s) | 400 expired | PASS | added during the run |

## Run history
| Date | Commit | Run | PASS | FAIL | BLOCKED | NOT RUN | Verdict |
|-------|--------|---------|------|------|---------|---------|-------|
| 2026-08-20 | abc1234 | 47 | 43 | 3 | 1 | 0 | NO-GO |
```

## ID rules

- `<PREFIX>-001` upward, zero-padded, **never renumbered**. A finding, a
  regression test and next month's re-run all point at the same ID.
- **The prefix is a short unique slug of the suite** (2–4 letters, declared at
  the top of the suite file, e.g. `KPN` for the coupon suite) — never a bare
  `TC`. With one suite per feature, unprefixed IDs collide across suites and
  every cross-reference (`known-issues.md`, regression tests, reports) becomes
  ambiguous. Check the `.qa/README.md` suite index for taken prefixes.
- New cases on a later run continue the sequence — don't reset, don't reuse the ID
  of a deleted case.
- Suffix `*` marks a case discovered *during* execution rather than designed up
  front. Worth keeping visible: a high `*` count means Phase 1 was too shallow,
  and that's useful feedback about your own design.
- Reference IDs everywhere: findings (`KPN-021 → BUG-017`), regression tests
  (`test_concurrent_redeem  # KPN-021`), the report's coverage table.

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

- **One assertion per case.** "Invalid e-mail is rejected *and* the error message
  is localised" is two cases; when it fails you want to know which half broke.
- **Concrete data, not descriptions.** `SAVE10`, `₺100`, `1,000 characters` — not
  "an invalid value". Someone else (or you, next month) must be able to re-run it
  without re-deriving the inputs.
- **Expected must be checkable and anchored.** Not "returns an error" but "400 +
  `code=COUPON_EXPIRED`, no order created". If nothing in the requirements or
  schema anchors your expectation, that's an open question, not a case.
- **Write money, quantity and counter deltas as the expected number**
  (`balance 1000 → 994`), never as the rule for computing it ("the refunded part
  goes back"). Leaving the arithmetic to the executor has two failure modes: they
  get it wrong and report a false FAIL, or they derive the number from what the
  system returned and certify a wrong result.
- **Include the verification point**, not just the request: which DB row, which
  log line, which counter you'll look at.
- **Steps short enough to repeat.** If a case needs 12 steps of setup, the
  precondition column is doing the wrong job — split it.

## Lifecycle across runs

1. **First run:** design the list, deliver it, execute it, append the run to
   `## Run history`.
2. **Later runs on the same area:** re-run the existing cases as the regression
   suite, then append new cases for whatever the new diff introduced. Mark cases
   that no longer apply as `SKIP-N/A` with a note — delete rather than leave a
   silently stale expectation only when the feature is genuinely gone.
3. **When a case keeps catching things**, promote it to an automated test and note
   the test's path in the row. The case list is the backlog of tests worth
   automating; the ones that never fail are cheap to keep as manual checks.
