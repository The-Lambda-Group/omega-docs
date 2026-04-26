[< Home](../README.md) | [Gotchas](README.md)

# call Inside with-table-if Scales Poorly

## Symptom

Calling an in-memory clause (via `call`) inside a `with-table-if` branch works for small inputs but hangs or crashes when the number of calls grows beyond ~15.

## Context

This was discovered during MAB Enumerate development. A `SortedConcat` in-memory clause was called inside a `with-table-if` else branch to accumulate sorted results. With 3-4 list entries (6-8 calls), it worked fine. At 5+ entries (~15+ calls), the query hung indefinitely.

## Root Cause (Suspected)

The `call` term solution fix (commit `5de5ffd` in omega-db) nests conjunctions into the solution structure. Each `call` adds another layer of nesting. At scale, this creates exponentially deeper solution structures that overwhelm the runtime.

This may be a performance issue (fixable with optimization) or a correctness bug (the conjunction nesting approach may be fundamentally wrong for repeated calls). Investigation in the omega-db REPL is needed.

## Workaround

For iterative accumulation patterns, avoid calling in-memory clauses inside `with-table-if` at scale. Options:

1. **Pre-compute outside the branch** — move the `call` before the `with-table-if` if possible.
2. **Use a different accumulation strategy** — fold or aggregate at the datastore level rather than calling clauses iteratively.
3. **Limit batch size** — if the pattern must be used, keep inputs small (under ~10 calls).

## Status

Open — needs investigation in omega-db REPL to determine whether it's a performance issue or a correctness bug.

## Related

- [Clauses — In-Memory Clauses](../reference/clauses.md#in-memory-clauses-bound-to-a-symbol) — the language-reference home for in-memory clauses (the kind invoked by `call`). Defines the `(:- ...)` and `clause` term forms that produce `call`-able symbols.
- [Clauses — Closures / Capture](../reference/clauses.md#closures--capture) — when the in-memory clause needs an outer-scope binding, capture is the syntax. Includes the warning about known edge cases with capturing clauses inside other captured clauses, which is relevant if you are nesting `call`s of captured helpers.
- [with-table-if header must include captured clause symbols](with-table-if-capture-header.md) — the scope-boundary rule: a captured clause symbol invoked inside `with-table-if` must appear in the header.
