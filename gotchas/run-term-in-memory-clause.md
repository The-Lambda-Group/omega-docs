[< Home](../README.md) | [Gotchas](README.md)

# run-term Does Not Work on In-Memory Clauses

## Symptom

Using `run-term` to invoke an in-memory clause produces a 500 error.

```oql
;; BAD — 500 error
(run-term MyClause arg1 arg2 Result)
```

## Why

`run-term` expects a functor object backed by a datastore — it looks up a clause definition from persistent storage. An in-memory clause (one bound in the current solution table, not stored in a datastore) is not a functor object and cannot be resolved by `run-term`.

## Fix

Use `call` instead. `call` is designed for in-memory clause invocation:

```oql
;; GOOD — call works with in-memory clauses
(call MyClause arg1 arg2 Result)
```

## When to Use Which

| Term | Use For |
|------|---------|
| `call` | In-memory clauses (symbols bound in the solution table) |
| `run-term` | Functor objects from datastores |
| `run-page` | Invoking a full page by ID through the public API |

## Related

- [Clauses — Stored vs In-Memory](../reference/clauses.md#in-memory-clauses-bound-to-a-symbol) — the language-reference distinction between stored clauses (callable by `run-term`) and in-memory clauses (callable by `call`).
- [Clauses — Closures / Capture](../reference/clauses.md#closures--capture) — when the helper must be a stored clause for `run-term` to find it but the body still needs an outer-scope binding, capture is the bridge: define the stored clause at the top level, build a functor, and capture the functor into the implementation clause that calls it.
- [`call` vs `run-term` solution scoping](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/explanation/call-vs-run-term-on-captured-helpers.md) — the deeper explanation of when each term is structurally required (per-row freshness, partition-before-fold) vs interchangeable.
