[< Home](../README.md) | [Gotchas](README.md)

# run-term Does Not Work on In-Memory Clauses

## The Rule

**For in-memory captured-helper invocations: use `call` by default. `run-term` is appropriate ONLY when it is replacing `with-group-by` — i.e., the canonical partition-before-fold scaling pattern.**

In every other context (one-solution-in / one-solution-out helpers, list-mapping helpers, validation helpers, transformation helpers, etc.), `(run-term ...)` against an in-memory captured helper is wrong. The hard failure mode is a 500 (described below). The softer, more dangerous failure mode is engine spin-out / `ECONNRESET` — a malformed `run-term` invocation that the engine cannot resolve cleanly can stall the database and require a restart.

If you are not replacing a `with-group-by`, you are not in the exception. Use `call`.

The reasoning chain: `run-term` runs its target as a separate execution against a fresh solution. That is mechanically what makes the partition-before-fold pattern fast (each invocation sees only one partition's rows). Per-row freshness — the other valid `run-term` use case discussed in [`call` vs `run-term` — solution scoping, freshness, and partition-before-fold](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/explanation/call-vs-run-term-on-captured-helpers.md) — requires a **stored** helper, not an in-memory captured one. So inside the captured-helper world, the partition replacement is the only valid `run-term` shape.

## Why agents get this wrong

Working impls in the codebase (e.g., the Manager page's `SortedConcat` / `SortInPartition` helpers, `BuildCrossProduct`'s sorted-concat) use `run-term` legitimately because they are replacing a `with-group-by`. An agent reading those impls and pattern-matching on "this codebase uses run-term for helpers" will conclude `run-term` is a general-purpose alternative to `call`. It is not. Every `run-term` you see in a working impl is doing partition replacement; if your helper is not doing that, copying the form will spin the engine out.

Concrete trigger that motivated this section (2026-04-27): a list-conversion helper (`CombinedList → list of JSON-stringified treatment_ids`) was invoked via `(run-term TupleListToTreatmentIds CombinedList TargetIds)`. The helper is one-solution-in / one-solution-out — exactly the case where `call` is correct. Result: ECONNRESET, engine spun out, database restart required.

## Symptom

Using `run-term` to invoke an in-memory clause produces a 500 error.

```oql
;; BAD — 500 error
(run-term MyClause arg1 arg2 Result)
```

In some shapes (notably when the in-memory helper is reached via capture and the args do not align cleanly), the failure is not a 500 but an `ECONNRESET` with the engine in a stalled state. Same root cause, harder failure mode.

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
| `call` | In-memory clauses (symbols bound in the solution table). **Default for captured-helper invocations.** |
| `run-term` | Functor objects from datastores. For captured-helper invocations, use **only** when replacing `with-group-by` (partition-before-fold scaling). |
| `run-page` | Invoking a full page by ID through the public API. |

## Iteration-context exception: per-row invocation at scale

There is a second valid `run-term` shape for captured in-memory helpers: **per-row invocation inside a multi-solution iteration at scale (K ≥ ~30 rows)**.

Empirically (MAB Allocate V1, 2026-04-30, K=40 arms): two independent `(call CapturedHelper ...)` per-row invocations both produced engine spin-out / hangs. Switching both to `(run-term CapturedHelper ...)` fixed the hangs and dropped wall-clock to 3–5s. The hypothesis is that `call` per-row shares solution scope with the enclosing K-row iteration, and accumulated residue scales non-linearly. `run-term` creates a fresh solution per invocation, isolating per-row state.

**The updated rule:**

| Shape | Use |
|-------|-----|
| One-solution-in / one-solution-out helper | `call` |
| List-mapping, validation, transformation helper | `call` |
| Replacing `with-group-by` (partition-before-fold) | `run-term` |
| Per-row invocation in multi-row iteration at K ≥ ~30 | `run-term` |

If you are iterating over K rows and calling a helper once per row where K might reach ~30+, use `run-term` even though the helper is an in-memory captured clause.

See [run-term per-row in multi-solution iteration](../explanation/run-term-per-row-iteration.md) for the full pattern, worked examples, scaling data, and cargo-cult sources.

## Related

- [Clauses — Stored vs In-Memory](../reference/clauses.md#in-memory-clauses-bound-to-a-symbol) — the language-reference distinction between stored clauses (callable by `run-term`) and in-memory clauses (callable by `call`).
- [Clauses — Closures / Capture](../reference/clauses.md#closures--capture) — when the helper must be a stored clause for `run-term` to find it but the body still needs an outer-scope binding, capture is the bridge: define the stored clause at the top level, build a functor, and capture the functor into the implementation clause that calls it.
- [Reducers — Performance: partitions and `run-term`](../reference/reducers.md#performance-partitions-and-run-term) — partition-before-fold: when replacing `with-group-by` with a per-partition `run-term` invocation for scaling, `run-term` is correct. The per-row-iteration exception uses the same fresh-solution mechanism.
- [`call` vs `run-term` solution scoping](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/explanation/call-vs-run-term-on-captured-helpers.md) — the deeper explanation of when each term is structurally required (per-row freshness, partition-before-fold) vs interchangeable. Note: the per-row-freshness case requires a **stored** helper; for captured **in-memory** helpers the two valid `run-term` shapes are partition replacement and per-row iteration at scale.
- [run-term per-row in multi-solution iteration](../explanation/run-term-per-row-iteration.md) — the iteration-context exception: empirical evidence, threshold data, hypothesis, and the updated rule table.
