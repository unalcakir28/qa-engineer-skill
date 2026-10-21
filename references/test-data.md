# Test data: isolation, cleanup, validity

Fixture isolation is a Phase 2 precondition; this file is the *how*. Everything
here is stack-agnostic — apply it through whatever ORM, HTTP client or seed
mechanism the project already uses.

## Naming: every record traceable to its run

Give every created record an identifiable marker — a prefix or a dedicated
field: `qa-<date>-<agent>-<seq>` in a name/code/e-mail field works everywhere.
Two payoffs: cleanup becomes a single delete-by-prefix, and when a stray record
corrupts a later run, its name tells you which run and which agent made it.

## Isolation levels — pick the cheapest that holds

1. **Per-case fixture** — the case creates what it mutates, asserts, done.
   Default for state-transition and mutation cases.
2. **Per-agent fixture set** — in parallel runs, each agent seeds its own
   tenant/account/parent records at start and works only inside them. Mandatory
   for parallel execution: two agents sharing one mutable record produce FAILs
   that look exactly like real bugs.
3. **Shared read-only fixtures** — reference data (lookup tables, config rows)
   can be shared *if no case in the run mutates it*. One mutation demotes it to
   level 1 or 2.
4. **Exclusive run** — anything touching global state (DDL, app config, shared
   counters, feature flags) runs alone, never alongside parallel agents.

## Mutation discipline

- Before designing, mark each case as *reads* or *mutates*. Mutating cases
  either restore what they changed or run against a fixture nothing else uses.
- "Restore" means restore, not approximate: reactivating a record you
  deactivated is easy to forget half of (dates, counters, derived totals).
  When restoration is fiddly, prefer a throwaway fixture instead.
- Before an overwrite or delete on a fixture you did not create, snapshot the
  original and check that the snapshot can be put back (a stored file still
  resolves, a referenced object still exists). An original that only exists
  behind a link the overwrite will break is not restorable. Use a throwaway
  fixture instead, or leave the case `NOT RUN`.
- **Aim a real write by a stable attribute, then read it back before
  submitting.** Select the field a write targets by something that names it (a
  `name`, a label, a test id), never by position or by a selector that assumes
  markup the app may not have: field order differs between record variants, and
  a positional pick lands on the neighbouring field without an error. Before a
  write to shared or real data goes out, read back what the system says will
  change (a review or diff screen, the outgoing request body) and compare it with
  the change that was approved. A mismatch stops the case before the write.
- After a run that mutated shared data unexpectedly, **re-seed rather than
  trust leftover state** — chasing FAILs caused by a previous scenario's
  footprints costs more than a fresh seed.

## Idempotent seeding

Seed scripts must be safe to run twice: upsert by a natural key instead of
blind insert, or delete-by-prefix first. A seed that duplicates rows on the
second run creates exactly the phantom-duplicate bugs you're testing for.

## Synthetic values must be *valid* synthetics

A synthetic value that fails format validation never reaches the business rule
you meant to test — you tested the validator instead, and every case downstream
reports a false rejection. Before generating test values, check the field's
format constraints (UUID, e-mail, IBAN/checksum formats, phone, enum, length)
and satisfy them. Corollary: when a whole group of cases fails with the same
rejection, suspect your synthetic data before the code.

**Read the value, not the identifier.** For anything whose wire form is defined
somewhere else — enums, scopes, status codes, feature keys, content types, error
codes — take the value from the definition (read the file, or resolve it at
runtime) instead of retyping the name you saw in code. A member named
`READ_ONLY` whose actual value is `read:only` fails validation exactly like a
typo, and the rejection is indistinguishable from a real bug. This is the single
most common source of harness-caused fake FAILs, and it survives the "valid
format" check above because the shape looks right.

## Time-sensitive data

Never rely on "now" landing on the right side of a boundary. Create records
with explicit dates relative to the boundary you're testing (expiry - 1 day,
expiry, expiry + 1 day), and remember timezone: the server's "today" may not be
the test machine's.

## Cleanup

- End the run with a delete-by-prefix sweep of what the run created — unless
  the user wants the data kept for inspection; ask when unsure.
- Never delete data you didn't create. A shared dev DB's pre-existing rows are
  someone else's work.
- Log what was left behind (counts, prefixes) in the report's cleanup line, so
  leftover data is a documented decision, not a surprise.

## Secrets in test data

- Never bake real credentials into fixtures, scripts or case files — reference
  the environment manifest's pointer to where they live.
- Test tokens are still tokens: they go in `.qa/evidence/` (git-ignored), never
  in committed suites, reports or seed scripts.
