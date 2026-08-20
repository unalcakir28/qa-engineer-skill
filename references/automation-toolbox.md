# Automation toolbox — machines that test better than a hand-written list

A case list is bounded by what you thought of. These tools search spaces you
can't enumerate by hand. Each block says what it catches, what it costs, and when
it's worth it — pick by situation, don't run all of them by reflex, and always
report what a tool found *or* that it found nothing.

Before installing anything: check what the project already has, follow its
conventions, and prefer a tool already in the dependency file. If a run needs a
new dev dependency, say so rather than adding it silently.

- [1. Spec fuzzing (best value for an API)](#1-spec-fuzzing)
- [2. Property-based testing](#2-property-based-testing)
- [3. Concurrency harness](#3-concurrency-harness)
- [4. DB invariant sweep](#4-db-invariant-sweep)
- [5. Golden / snapshot tests](#5-golden--snapshot-tests)
- [6. Load smoke with thresholds](#6-load-smoke-with-thresholds)
- [7. Fault injection](#7-fault-injection)
- [8. Mutation testing](#8-mutation-testing)
- [9. Security baseline checks](#9-security-baseline-checks)
- [10. Accessibility & localisation passes](#10-accessibility--localisation-passes)

---

## 1. Spec fuzzing

**Catches:** unhandled input → 500s, missing validation, responses that violate
the schema the API advertises, auth gaps, stateful sequence bugs. Generated from
the OpenAPI/GraphQL schema, so it costs no case-writing.

**Tools:** `schemathesis` (OpenAPI/GraphQL, the practical default), RESTler for
stateful multi-step fuzzing, `dredd` for plain spec conformance.

```
schemathesis run <openapi-url-or-file> --header "Authorization: Bearer <key>" --checks all
```

**Worth it:** yes, first choice for any API product — minutes to set up, typically
surfaces a handful of real issues on a schema's first run. Run it twice with two
different tenants' keys and you get a tenancy check for free.

**Report it as:** individual findings (each reproducible request), not "fuzzer
found 12 things".

## 2. Property-based testing

**Catches:** the input classes nobody writes by hand — empty, unicode, huge,
negative zero, deeply nested, round-trip breaks — plus sequence bugs via stateful
mode (e.g. paginate-while-deleting).

**Tools:** Hypothesis (Python), fast-check (JS/TS), FsCheck (.NET).

**Where to point it:** validators, parsers, serialisers, money and date maths,
slug/normalisation functions, anything with an invariant you can state in one
sentence ("parse(format(x)) == x", "total never negative", "sorted output is a
permutation of input").

**Worth it:** yes — hours of effort, and it keeps finding things after you stop
thinking about it.

## 3. Concurrency harness

**Catches:** double charges, oversold stock, quota bypass, duplicate records
behind one idempotency key, last-write-wins overwrites.

**How:** fire N identical requests as simultaneously as the runtime allows
(`asyncio.gather` + `httpx`, `Promise.all`, `Task.WhenAll`), with a barrier so
they dispatch together rather than in a loop — a naive sequential loop usually
misses the window. Then assert **exactly one** side effect: one row, one charge,
one decrement.

```python
async with httpx.AsyncClient() as c:
    await asyncio.gather(*[c.post(url, json=body, headers=h) for _ in range(20)])
# then: SELECT count(*) ... == 1
```

**Worth it:** yes, but scoped — payment, balance, quota, stock, invite/redeem,
anything with an idempotency key. Not the whole API.

## 4. DB invariant sweep

**Catches:** silent corruption that no single case looks for. Run at the end of
execution, over whatever the run touched.

Queries worth keeping in the project (adapt names):

```sql
-- orphans
SELECT count(*) FROM order_items oi LEFT JOIN orders o ON o.id=oi.order_id WHERE o.id IS NULL;
-- tenancy holes
SELECT count(*) FROM <table> WHERE tenant_id IS NULL;
-- impossible values
SELECT count(*) FROM accounts WHERE balance < 0;
SELECT count(*) FROM order_items WHERE quantity <= 0 OR unit_price < 0;
-- totals that don't reconcile
SELECT o.id FROM orders o JOIN order_items i ON i.order_id=o.id
GROUP BY o.id, o.total HAVING o.total <> SUM(i.quantity*i.unit_price) - o.discount;
-- soft-delete leaks
SELECT count(*) FROM <table> WHERE deleted_at IS NOT NULL AND <visible in some view>;
```

**Worth it:** yes — minutes, and non-zero results are almost always real bugs.

## 5. Golden / snapshot tests

**Catches:** regressions in complex output — reports, exports, generated
documents, aggregate payloads — where eyeballing a diff by hand is hopeless.

**Tools:** `syrupy`/`pytest-snapshot`, jest snapshots, Verify (.NET).

**Caution:** a snapshot freezes *current* behaviour, which collides with
non-negotiable #4. Only snapshot output you have verified is correct once, by
hand, and review every future diff rather than blessing it.

**Worth it:** yes for report/export endpoints; no for ordinary CRUD responses.

## 6. Load smoke with thresholds

**Catches:** N+1 queries, a missing index on a new filter column, unbounded
queries, connection-pool limits. You don't need production-scale infrastructure —
the signal is *relative regression* on your own machine.

**Tools:** k6 (thresholds as code, good CI story), Locust.

```js
export const options = { vus: 5, duration: '30s',
  thresholds: { http_req_failed: ['rate<0.01'], http_req_duration: ['p(95)<800'] } };
```

Pair it with the query log or `EXPLAIN` on the slowest endpoint — the count of
queries per request is often the more damning number.

**Worth it:** yes at L3 for the endpoints the change touched.

## 7. Fault injection

**Catches:** missing timeouts, retry storms, connection-pool exhaustion,
half-committed state when a dependency dies mid-operation.

**Tools:** toxiproxy between the app and its DB/cache/upstream (latency, timeout,
reset_peer, down); or simply stop the container.

**Worth it:** only for critical paths — do it once against the DB and the payment
or main upstream, then keep the recipe in `.qa/`.

## 8. Mutation testing

**Catches:** tests that execute code without asserting anything meaningful. It
injects small faults (`<` → `<=`, remove a line, change a constant) and reports
which ones no test noticed. Line coverage says a line ran; a surviving mutant says
nobody checked what it did.

**Tools:** mutmut (Python), Stryker / StrykerJS (Node/TS), Stryker.NET.

**Worth it:** scoped to the changed module before a release — whole-repo runs are
too slow. Use it as the honest answer to "is this area actually tested?", and
turn surviving mutants into new cases.

## 9. Security baseline checks

The OWASP API Security Top 10 items a QA agent can actually verify, mapped to
what to do (report findings; don't escalate into exploitation):

| Item | Check |
|------|-------|
| **BOLA / IDOR** | With tenant A's key, request tenant B's object ids — expect 403/404. Also nested routes both ways. |
| **Broken authentication** | Expired/revoked/tampered key, key from another environment, key in URL vs header, no key, key after rotation. |
| **Object property level authz** | Mass assignment: send `role`, `isAdmin`, `price`, `tenantId`, `createdAt` in a create/update body and verify they're ignored. |
| **Resource consumption** | Oversized payload, `pageSize=100000`, no rate limit, expensive query loop, huge file. |
| **Function level authz** | Non-admin key against admin endpoints, and the less-loved verbs: `PATCH`, `DELETE`, bulk, export. |
| **Sensitive business flows** | Automated abuse of signup/invite/coupon-redeem — is there any throttle? |
| **SSRF** | Any "fetch this URL" surface (webhooks, avatar-from-URL, import): point it at `localhost`, `169.254.169.254`, internal hostnames. |
| **Misconfiguration** | CORS `*` with credentials, verbose errors/stack traces, debug routes, default creds, TLS/headers, exposed `/swagger` or `/actuator`. |
| **Inventory** | Live routes vs. the documented spec; old API versions still reachable. |

Tools that help: schemathesis with two tenants' keys, OWASP ZAP baseline scan,
`testssl.sh` for TLS. Deep authorisation-logic review and full ASVS/WSTG
compliance still need a human — say so.

## 10. Accessibility & localisation passes

- **axe-core** (via Playwright/CLI) on the changed pages: mechanical, catches a
  meaningful share of real issues — but roughly a third of WCAG criteria are
  automatable at all, so keyboard order, focus visibility and screen-reader
  experience remain a human check. State that limit rather than implying a clean
  scan means accessible.
- **Pseudo-localisation:** inflate every string ~40% and add accented characters
  to catch truncation and hardcoded widths; then check RTL mirroring if you ship
  Arabic/Hebrew.
- **Format checks:** Turkish decimal comma, thousands separator, `dd.MM.yyyy`,
  currency placement, locale-aware collation and case (`i`/`İ`), timezone of the
  viewer vs. the server.
