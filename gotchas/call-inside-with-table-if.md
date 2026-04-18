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
