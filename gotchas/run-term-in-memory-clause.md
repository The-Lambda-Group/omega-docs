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
