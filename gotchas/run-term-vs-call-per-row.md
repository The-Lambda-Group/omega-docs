[< Home](../README.md) | [Gotchas](README.md)

# run-term Forces a Fresh Solution Per Row; call Does Not

## Symptom

A clause invoked per-row via `call` (or via direct solution-row expansion) that generates a UUID or other non-deterministic value returns the **same** value for every row. When those values are used as primary keys for a write, the write fails with `KEY_UPDATE_CONFLICT` — the engine sees multiple writes to the same key.

Example observed in MAB Strategist (2026-04-19):

```oql
;; Called across 3 row pages — expected 3 distinct block-ids
(Qo.Public.OqlApi.Page.Bl.Html/add-or-get-by-name
    FolderId RowPageId Name Html BlockId)
```

All three invocations returned the **same** `BlockId`:
`0d6fc01a-7ed4-4f27-9c2e-b24d9c9ba484`. Writing three different
`RowPageId` values against one shared `BlockId` violated the page-block
primary key and the whole query failed with `KEY_UPDATE_CONFLICT`.

## Why

OQL clause bodies execute against a single solution context. When a clause
(either stored or in-memory) is invoked across N solution rows by using
`call` or by letting the solution table expand the invocation directly,
the engine does **not** re-execute the body N times from scratch. It
executes the body once and joins the resulting solution rows back in.

Non-deterministic built-ins like `(uuid Id)` are evaluated **once during
that single execution**. Every row receives the same bound value. This is
consistent with OQL's logic-language semantics: `uuid` is treated as a
relation (Id is bound to some fresh value), and the binding is shared
across the join just like any other binding.

`run-term`, by contrast, invokes a stored functor by looking it up in its
datastore and running the clause body in a **fresh solution** for each
invocation. Because each invocation is a separate execution, `uuid` fires
fresh each time — every row gets a distinct value.

## Fix

Wrap the per-row work in a stored helper clause and invoke it via
`run-term` inside the iteration. `run-term` forces a fresh solution per
invocation, so `uuid` (and any other non-deterministic built-in) produces
a new value each time.

```oql
;; BAD — one solution, one uuid shared across all rows
(Qo.Public.OqlApi.Page.Bl.Html/add-or-get-by-name
    FolderId RowPageId Name Html BlockId)
(Qo.Public.OqlApi.Page.Bl/write! FolderId RowPageId BlockId Block)
;; On 3 rows: all three writes use the SAME BlockId → KEY_UPDATE_CONFLICT
```

```oql
;; GOOD — helper clause invoked via run-term forces a fresh solution per row
(:- (MyNs/write-html-block! FolderId RowPageId Name Html _)
    (Qo.Public.OqlApi.Page.Bl.Html/add-or-get-by-name
        FolderId RowPageId Name Html BlockId)
    (Qo.Public.OqlApi.Page.Bl/write! FolderId RowPageId BlockId _))

;; Per-row invocation — uuid fires fresh each time
(run-term MyNs/write-html-block! FolderId RowPageId Name Html _)
```

The enumerate implementation's `CanNotCombine` helper uses this same
pattern to guarantee distinct identity per row.

## When to Use Which

| Term | Behavior | Use For |
|------|----------|---------|
| `call` | One solution, shared bindings across all rows | Pure transformations where the same binding per row is fine or intended |
| `run-term` | Fresh solution per invocation, non-deterministic built-ins fire fresh | Per-row writes, UUID generation, per-row side effects, any case where each row must have its own identity |
| `run-page` | Fresh query execution through the public API | Invoking a full page by ID |

## Related

- [run-term does not work on in-memory clauses](run-term-in-memory-clause.md) — `run-term` requires a stored functor; you cannot use it on an in-memory clause. Wrap the logic in a stored `(:- (Ns/name! ...) ...)` clause first.
- [OQL Execution Model — Clauses as Solution Boundaries](../explanation/oql-execution-model.md#clauses-as-solution-boundaries) — why clause invocations share a solution and what that implies for bindings.
