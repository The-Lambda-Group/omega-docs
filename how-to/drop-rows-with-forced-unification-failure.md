[< Home](../README.md) | [How-to](README.md)

# Drop rows with forced unification failure

How to deliberately exclude rows from a solution table in OQL using the `(= true false)` idiom.

## When to use this

Any time you want a `with-table-if` branch to drop rows rather than write them. Typical use cases:

- Filtering out HTTP responses that shouldn't be processed (e.g. 422s — see [Handle HTTP 422 in dispatch](handle-http-422-in-dispatch.md))
- Excluding rows that match a sentinel value from an otherwise-writable solution
- Short-circuiting a branch during debugging

## The idiom

```oql
(with-table-if [RowId HttpStatus] (= HttpStatus 422)
  (= true false))
```

`(= true false)` is a unification that cannot succeed. The solver records failure and the row is excluded from the downstream table.

## Why this works

OQL's solution model is relational: rows that fail unification are simply not in the solution. `(= true false)` is the shortest failing unification — any forced-fail expression works, but `(= true false)` is the idiom because it reads as "make this branch unsatisfiable."

## Anti-patterns

- Don't do `(= true false)` in a `with-table-if` *else*-branch — the else fires when the condition is false, and you usually want the else to pass rows through, not drop them.
- Don't do it in the top-level body of a query — you'll drop the entire solution.
- Don't use it where you actually want to raise an error. `(throw ...)` is the error primitive; `(= true false)` is a silent drop.
