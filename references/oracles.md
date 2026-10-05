# Oracles: deciding what "correct" means

A test needs two things: a scenario and a way to know whether the result was
right. The second one is the **oracle**, and it's the part that quietly gets
skipped — an agent that only checks "did it crash" is testing almost nothing.

Often a spec, task document or plan already exists — frequently in the very
session that produced the code. That is your first-class oracle: read it, extract
each acceptance criterion as a basis item, and test against what was asked for,
not against what the code happens to do. Note that a spec is itself fallible: if
it contradicts the schema, skips the error paths, or leaves a rule ambiguous, that
gap is a finding (an open question), not something to paper over.

The heuristics below are for everything the spec doesn't cover — and there is
always a remainder, because specs describe the happy intent and rarely the edges.
Working testers use a set of consistency heuristics (Bolton & Bach's
**FEW HICCUPPS**): a behaviour is suspect when it's inconsistent with something it
ought to be consistent with. Name the one you used — it turns "I think this is
wrong" into an argument the developer can act on.

## The heuristics

| Oracle | Ask | Where to look in a codebase |
|--------|-----|------------------------------|
| **History** | Did this behave differently before? Was the change intended? | `git log -p`, the previous release, changelog, old tests |
| **Image** | Would a user think worse of the product for this? | error copy, ALL-CAPS stack traces shown to users, dead ends |
| **Comparable products** | How do similar systems behave here? | how Stripe/GitHub/any peer API handles the same case (pagination, error codes, idempotency) |
| **Claims** | Does it match what we say it does? | README, API docs, OpenAPI descriptions, UI labels and help text, the PR description, the ticket |
| **User desires** | Would a reasonable user expect this? | support requests, TODOs, the feature's purpose |
| **Product** | Is it consistent with the rest of this system? | do sibling endpoints validate, page, error, sort, and name fields the same way? |
| **Purpose** | Does it serve what the feature exists to do? | the ticket's motivation, not just its acceptance criteria |
| **Statutes** | Any legal/regulatory rule? | invoicing rules, KVKK/GDPR (personal data in logs, deletion, retention), tax/VAT |
| **Familiarity** | Does it look like a bug pattern we've seen before? | `.qa/known-issues.md`, classic off-by-one / N+1 / race shapes |
| **Explainability** | Can I explain why it does this? | if nobody can explain the behaviour, it's a finding even if it might be correct |
| **World** | Does it make sense against plain reality? | negative quantities, orders shipped before payment, ages of 700, 31 February |

Two more that are mechanical and worth automating:

- **Consistency within the data**: the same fact computed two ways must agree —
  line items vs. order total, the counter vs. `COUNT(*)`, the list response vs.
  the export.
- **Consistency across time**: run the same operation twice; an idempotent action
  must not change state the second time.
- **The project's own conventions over textbook ones**: before writing an
  expected status code, error shape or empty-result form, check how the project's
  error layer and its sibling endpoints already answer the same situation (a
  "not found" that every endpoint returns as a generic client error, not as a
  dedicated status). An expectation copied from a style guide the project never
  adopted produces false FAILs. A deviation from the project's own convention is
  the finding; a deviation from the textbook is at most an open question.

## Metamorphic relations — an oracle when you can't know the exact answer

When the correct output is hard to compute (search ranking, pricing, reports),
don't guess the value; assert a *relation* that must hold:

- Adding a filter can never *increase* the result count.
- Sorting the input must not change the set of results, only their order.
- Calling an idempotent endpoint twice yields the same state as calling it once.
- Doubling every quantity must double the subtotal (before rounding rules).
- The sum of paginated pages equals the unpaginated total.

These are cheap to write, need no ground truth, and catch real logic bugs.

## Test basis and traceability

Write down what each case is checked against — the **basis** — and keep it in the
case list's `Basis` column:

- `ticket §3` — an explicit acceptance criterion
- `schema: unique(tenant_id, email)` — a constraint
- `docs: /api/orders` — a documented promise
- `oracle: history` — the previous version behaved differently
- `oracle: product` — sibling endpoints do it another way

Then use it in both directions:

1. **Case → basis:** an expectation with no basis is an *open question*, not a
   bug. Ask instead of guessing.
2. **Basis → case:** list the basis items (each acceptance criterion, each
   documented promise, each constraint) and check every one has at least one case
   pointing at it. The ones with none are your coverage holes — report them, even
   the ones you deliberately left untested.

That second direction is what a naive sweep never does, and it's the difference
between "I ran 47 cases" and "every requirement is covered except these three,
and here's why".
