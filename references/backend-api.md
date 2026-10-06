# Backend / API / data-layer checklist

Merge the applicable items into the Phase 1 matrix. Not every line applies to
every endpoint — but each line you skip should be skipped knowingly.

## Contract and HTTP semantics

- Status codes: 200 vs 201 (+ `Location`) vs 204; 400 vs 401 vs 403 vs 404 vs
  409 vs 422 vs 429; a validation failure returning 500 is a finding.
- Response shape matches the documented schema/OpenAPI: field names, types,
  nullability, date format, enum values, number vs string ids.
- Error responses: consistent envelope, machine-readable code, no stack trace,
  no SQL, no internal hostname, no PII in the message.
- Verbs: `GET` has no side effects; `PUT` is idempotent; `DELETE` twice returns
  the same thing; `HEAD`/`OPTIONS` don't blow up; unsupported method → 405.
- Headers: missing/wrong `Content-Type`, missing `Accept`, absent
  `Authorization`, malformed bearer token, huge header, unexpected charset.
- Compatibility: existing clients still work — no removed field, no renamed key,
  no narrowed type, no newly-required request field.
- Generated clients: if an SDK is generated from the contract, a new optional
  parameter inserted ahead of existing ones shifts positional calls. Check the
  parameter order the contract emits, not only that the change is additive.

## Input validation

- Every field: missing, `null`, empty string, whitespace-only, wrong type,
  wrong format, too long, out of range, wrong enum member, wrong case
  (`"ACTIVE"` vs `"active"`).
- Unknown/extra fields: ignored or rejected — but never silently persisted.
- **Mass assignment:** send `id`, `userId`, `tenantId`, `role`, `isAdmin`,
  `status`, `price`, `balance`, `createdAt` in the body and check none of them
  takes effect.
- Type juggling: `"5"` vs `5`, `"true"` vs `true`, `[]` vs `{}` vs `null`,
  array where scalar expected, nested object 20 levels deep.
- Body edge cases: empty body, malformed JSON, duplicate keys, trailing comma,
  BOM, 10 MB payload, 10 000-element array.
- Sanitisation vs validation: is trimming/normalising applied consistently on
  write, read, search and uniqueness checks?

## Authentication and authorisation

- Anonymous, expired token, tampered signature, token from another environment,
  token for a deleted/suspended user, valid token missing the required scope.
- **IDOR:** as user A, request/update/delete B's resource by guessing the id
  (sequential ids, or an id captured from another account). Also nested routes:
  `/orgs/{otherOrg}/items/{myItem}` and `/orgs/{myOrg}/items/{otherOrgItem}`.
- Full role × action grid, including `PATCH`, bulk endpoints, export/download,
  and any "internal" endpoint that forgot its guard.
- Privilege escalation: can a user grant themselves a role, add themselves to
  another tenant, invite to an org they don't own, or approve their own request?
- Multi-tenancy leakage in *lists*, search, counts, aggregates, exports and
  autocomplete — the single-record fetch is usually guarded, the list is not.
- 403 vs 404 consistency (existence leakage) and error-message enumeration
  (login/reset flows telling you whether an email exists).

## Business rules and calculations

- Re-derive every computed number independently (by hand or a small script) and
  compare: totals, discounts, tax, commission, VAT, shipping, proration,
  currency conversion, interest, scoring.
- Rounding: half-up vs banker's, per-line vs per-total, float vs decimal —
  `0.1 + 0.2`, `1.005`, three-decimal currencies, negative amounts.
- Quantity/stock/balance rules: exactly-at-limit, one over, going negative,
  reservations that expire, refunds larger than the payment.
- Discount/coupon logic: expired, not-yet-valid, wrong customer, already used,
  used twice concurrently, stacked with another, applied to an empty cart, 100%
  discount, discount > total.
- Dates: end before start, same instant, DST jump, timezone of the *user* vs
  server vs DB, "today" near midnight, leap day, month-end arithmetic
  (`Jan 31 + 1 month`), age/duration off-by-one.

## Persistence and data integrity

- After every mutation, read the source of truth (the DB), not the response echo.
- Constraints actually exist in the schema, not only in code: unique, FK,
  not-null, check. Try to violate each directly.
- Transactions: force a failure halfway (invalid second row, killed process) and
  verify full rollback — no half-created aggregate, no orphan child, no counter
  incremented without its row.
- Soft delete: deleted rows disappear from lists, search, counts, exports,
  relations and unique checks — and can't be fetched by id.
- Cascades: deleting the parent does the intended thing (and not the
  unintended one). Reusing a deleted record's unique value.
- Migrations: run forward on a copy with realistic data; check defaults for
  existing rows, nullability, index presence, and whether the down/rollback
  path exists and works.
- Audit/timestamps: `createdAt`/`updatedAt`/`createdBy` set correctly, not
  overwritable by the client, and not silently mutated on unrelated updates.
- Encoding/collation: Turkish characters and emoji survive a round trip; does
  `i` match `İ` in uniqueness or search where it shouldn't (or should)?

## Concurrency and idempotency

- Double submit the same create (two identical requests within milliseconds):
  one record or two?
- Two writers on one row: last-write-wins silently overwriting a field, or
  optimistic-concurrency conflict properly reported?
- Parallel decrements of a shared counter/stock/balance (fire 20 concurrent
  requests): does it go negative or oversell?
- Retry after a client timeout where the server actually succeeded — is there an
  idempotency key, and is it honoured?
- Duplicate/out-of-order webhook or queue message; message processed twice;
  message processed after a later one.
- Locks: deadlock risk between two orderings, lock held across an external call,
  a job and a user request touching the same row.
- Scheduled jobs overlapping their own previous run.

## Lists, search, pagination

- `page`/`offset` 0, negative, beyond the last page, non-numeric; `pageSize` 0,
  1, max, max+1, huge.
- Stable ordering — the same item must not appear on two pages; check paging
  while data is being inserted.
- Filters combined, filters conflicting, unknown filter param, empty result,
  exactly-one result, filter injection into the ORDER BY / raw SQL.
- Search: partial match, case, accents and Turkish characters, special
  characters (`%`, `_`, `*`, quotes), very long query, empty query, SQL/NoSQL
  payloads.
- Counts and aggregates agree with the rows actually returned (and respect
  tenancy/soft delete).
- Performance sanity: N+1 queries, missing index on the new filter column,
  unbounded query with no limit — check the query log or `EXPLAIN` if easy.

## Failure, resilience and side effects

- Dependency down / slow / 500 / garbage / empty body: is there a timeout, a
  retry policy, a circuit breaker, and a sane user-facing error?
- Partial failure of a multi-step operation: is the system left consistent, and
  is the operation safely resumable/retryable?
- Side effects fire exactly once: email/SMS/push, webhook, payment capture,
  file write, cache invalidation, search index update, event published.
- Logging and observability: the failure is logged with enough context, and
  logs contain no secrets, tokens, card numbers or personal data.
- Rate limiting: the request that hits the limit, the one after, and whether the
  limiter is per-user or global (and bypassable by changing a header).

## Security probes (report, don't weaponise)

- Injection: SQL/NoSQL/command/template payloads in every string field and every
  query param, including sort and filter names.
- Path traversal (`../../etc/passwd`) in any file/path/name parameter; upload of
  a wrong-type or oversized file; content-type spoofing.
- Stored payload that becomes XSS when a consumer renders it.
- Secrets exposure: config/`.env` endpoints, debug routes, verbose errors,
  directory listing, source maps, `/actuator`, `/swagger` in a protected env.
- CORS wide open, missing auth on an internal-only route, JWT `alg: none`,
  unsigned/unexpired tokens, session not invalidated on logout or password
  change.
