# Test design techniques

Read this in Phase 1. The point of a technique is that it produces cases you
would not have improvised. Pick the ones that fit the feature — you rarely need
all of them — and note in the report which you used.

- [1. Equivalence partitioning](#1-equivalence-partitioning)
- [2. Boundary value analysis](#2-boundary-value-analysis)
- [3. Decision table](#3-decision-table)
- [4. State transition testing](#4-state-transition-testing)
- [5. Pairwise / combinatorial reduction](#5-pairwise--combinatorial-reduction)
- [6. CRUD + permission grid](#6-crud--permission-grid)
- [7. Error guessing / attack heuristics](#7-error-guessing--attack-heuristics)
- [8. Cause–effect and data flow](#8-causeeffect-and-data-flow)
- [9. Exploratory charters](#9-exploratory-charters)
- [10. Risk-based prioritisation](#10-risk-based-prioritisation)
- [11. Differential (A/B) execution against the pre-change build](#11-differential-ab-execution-against-the-pre-change-build)

---

## 1. Equivalence partitioning

Split every input into classes that the system should treat identically, then
test **one** value per class — plus one from each *invalid* class, which is where
the bugs are.

For a `quantity` field (1–100, integer):
valid `{1..100}` → test `42`; invalid classes: `0`, negative, `>100`, decimal,
non-numeric, empty, `null`, whitespace, scientific notation (`1e2`), leading zeros
(`007`), thousand separators (`1.000`), hex-ish (`0x10`).

Do this per field, then note that *class combinations* across fields are what
Pairwise (§5) is for.

## 2. Boundary value analysis

Bugs cluster at edges because `<` vs `<=` is a one-character mistake. For each
bound test **min-1, min, min+1, max-1, max, max+1**.

Where to look for bounds: string length limits (and DB column widths — a
`varchar(50)` is a bound even if no validator mentions it), numeric ranges,
dates (start/end of validity, today, yesterday, expiry moment exactly),
pagination (`page=0`, `page=1`, last page, last page+1, `pageSize=0`, `1`, max,
max+1), collection sizes (0 items, 1 item, exactly the page size, page size+1),
file sizes/counts, quotas and rate limits (the request that hits the limit
exactly, and the next one), money precision (0.01, 0.001, and the rounding
boundary), and time (00:00:00, 23:59:59, month/year rollover).

## 3. Decision table

When behaviour depends on several conditions, tabulate instead of guessing.
Conditions as rows, rules as columns; one test per rule — including the
combinations the developer probably never tried.

| Condition | R1 | R2 | R3 | R4 |
|---|---|---|---|---|
| Logged in | Y | Y | Y | N |
| Owns record | Y | N | N | – |
| Record active | Y | Y | N | – |
| **Expected** | 200 | 403 | 409 | 401 |

Explicitly hunt for the "impossible" combinations — if a combination is supposed
to be unreachable, try to reach it anyway (§7); reaching it is a finding.

## 4. State transition testing

Draw the state machine (order: `draft → pending → paid → shipped → delivered`,
plus `cancelled`, `refunded`, `expired`). Then build the full transition matrix:
states × events.

- Every **legal** transition: does it happen, and are side effects (stock, email,
  balance, audit log) correct exactly once?
- Every **illegal** transition: `cancel` a delivered order, `pay` a cancelled
  one, `approve` twice, edit after lock, refund more than paid. Expect a clean
  rejection with unchanged state — not a 500, and not a silent success.
- **Sequence effects:** A→B→A, do-undo-do, and the same event twice in a row.
- **Timing:** act on a record exactly at expiry; let a session/token expire
  mid-flow.

## 5. Pairwise / combinatorial reduction

With 4 fields of 4 options each, exhaustive is 256 cases; pairwise covers every
*pair* of values in ~16–20 and catches most interaction bugs. Build it greedily:
list the factors and levels, then fill rows so each new row covers as many
uncovered pairs as possible.

Typical factors: user role × record state × payment method × currency ×
locale × device. Keep any combination that is business-critical or known-fragile
as an explicit case on top of the pairwise set.

## 6. CRUD + permission grid

For every resource the feature touches, cross **actions** (create, read, list,
update, patch, delete, export, bulk) with **actors** (owner, same-tenant peer,
other-tenant user, each role, admin, anonymous, deactivated/suspended user,
expired token, valid token missing the scope, service account).

Every cell needs an expected verdict. Two failure modes to hunt:

- **Missing check** — other-tenant user succeeds (IDOR), or a role without the
  permission gets through on one of the less-used verbs (`PATCH` and bulk
  endpoints are usually the weakest).
- **Wrong failure** — returns 404 where it should 403 or vice versa, or leaks
  existence/ownership through differing error messages or response times.

## 7. Error guessing / attack heuristics

Deliberate mischief, informed by where systems usually break:

- Skip a step: call step 3's endpoint without doing step 1; deep-link into a
  wizard; POST what the disabled button would have sent.
- Bypass the client: send values the form validates away; remove a required
  field; add fields the API didn't ask for (`role`, `isAdmin`, `price`,
  `createdAt`, `id`) to test mass assignment.
- Repeat and interleave: double-click, resend the same request with the same
  idempotency key, replay an old request, two tabs on the same record.
- Break the middle: kill the process mid-transaction, timeout the dependency,
  return a 500/garbage/empty body from the upstream, unplug the network after
  the request but before the response.
- Wrong-shape data: array where object expected, string `"5"` where number
  expected, `null` where object expected, deeply nested payload, duplicate JSON
  keys, wrong `Content-Type`, empty body, oversized body.
- Stale everything: expired token, stale ETag/version field, cached list after
  a delete, a record deleted in another tab.

## 8. Cause–effect and data flow

Follow one piece of data end to end and check it at every hop: input →
validation → transformation → persistence → read model → cache → UI → export.
The classic bugs live in the mismatches: trimmed on write but not on search,
rounded on display but not in the total, timezone-converted on read but not on
filter, encoded once on save and twice on render, filtered in the list query but
not in the export.

For every write, verify **all** downstream effects, and verify each happens
exactly once: DB row, related rows, counters/aggregates, cache invalidation,
search index, audit log, notification/webhook, file storage.

## 9. Exploratory charters

Timeboxed, goal-directed poking — the technique that finds what the matrix
didn't imagine. Write charters, not free clicking:

> Explore **checkout with a nearly-expired session** using **slow network +
> double submit** to discover **duplicate charges or lost orders**.

Run 3–5 charters, ~10 minutes each, over the riskiest areas from Phase 0. Log
what you did, what you saw, questions raised, and promote anything reproducible
into a matrix case.

## 10. Risk-based prioritisation

You will not test everything, so choose deliberately. Score each area
`impact × likelihood`:

- **Impact**: money, data loss, security/privacy, legal, blocks all users,
  silent-and-wrong (worst kind), cosmetic.
- **Likelihood**: newly written, rewritten, complex conditionals, concurrency,
  many callers, weak existing tests, previously buggy, cross-team boundary,
  third-party integration.

Spend depth on the top of that list, breadth (one case per class) on the rest,
and say in the report what you deliberately deprioritised. Untested-and-declared
is a legitimate result; untested-and-implied-tested is not.

## 11. Differential (A/B) execution against the pre-change build

When the change under test can also be built and run in its **pre-change** form,
run both versions side by side against the *same* data store and the same
inputs. This converts the hardest question a run has to answer — *did my change
cause this, or was it always broken?* — from an argument into a measurement.

Setup: the current build on one instance, the baseline (merge base, previous
tag, or whatever is deployed today) on another, both pointed at one shared
database or fixture set. Every probe worth attributing is then issued twice.

What it buys, in order of value:

- **Attribution.** A finding that reproduces identically on the baseline is
  pre-existing *with evidence* instead of assertion — and the verdict rules in
  `release-gate.md` treat those two cases differently. A finding that appears
  only on the new build is a regression and outranks everything else.
- **A regression sweep you didn't have to design.** Byte-comparing the responses
  of surfaces the change was not supposed to touch (public contract output,
  reports and exports, error bodies, downstream feeds) is a far stronger claim
  than "I looked and it seemed the same".
  Include what no response shows: read back the columns the persistence layer
  maintains on its own — auto-stamped "updated at" times, version counters,
  trigger-maintained fields — on both builds. A new write path that touches
  them changes every report, sort and filter built on them, and no response
  diff will reveal it.
- **A red-green proof for the change itself.** The baseline *is* the pre-fix
  code, so the probe that fails there and passes here satisfies non-negotiable
  #3 without writing a throwaway test first.
- **Proof that the harness can see the defect at all.** This is the half that
  matters most for a concurrency, locking or isolation claim (non-negotiable #9):
  a green concurrent run against the new build means nothing until the *same*
  harness, unchanged, has been shown to go red against the baseline. Until then
  "no overshoot" and "my parallelism never actually collided" produce identical
  output. Run the baseline first, and treat its red as the licence to believe the
  green.

**Validate the baseline before trusting a single A/B result.** A baseline that
silently already contains the change proves the exact opposite of what you will
conclude from it. Confirm it independently of what you were told — the commit it
was built from, or a build/process start time that predates the first edit — and
write that proof into the report. Every "pre-existing" label in it rests on that
one fact.

Three hygiene rules: both instances must share the data store (otherwise you are
comparing fixtures, not code); mutating probes run against throwaway
fixtures, because a write issued to the baseline lands in the same database the
new build reads; and **asynchronous work breaks "side by side"**. If both builds
attach to the same queue, broker, outbox table or scheduler, a job one build
enqueues is consumed by whichever worker grabs it first, and the result is
attributed to the wrong code. For any probe whose outcome is decided off the
request path, run the builds **one at a time** — stop one before starting the
other — or give each its own broker namespace, and say in the report which you
did. Synchronous probes can still run in parallel.

It does not always apply — a brand-new feature has nothing to compare against,
the two versions may not be able to share a schema, and running old code against
a migrated database can corrupt it. Say which of those ruled it out; don't drop
it silently. Reach for it whenever the change touches shared behaviour, a
security boundary, or anything where "pre-existing" and "I broke it" lead to
different release decisions.
