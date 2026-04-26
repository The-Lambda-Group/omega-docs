[< Home](../README.md) | [Gotchas](README.md)

# add-or-get-by-name Binds the Same Block Across Multiple Solution Rows

## Symptom

Calling `Qo.Public.OqlApi.Page.Bl.Html/add-or-get-by-name!` across N per-row `Page` arguments returns the **same** `Block` (same `block-id`) for every row. Subsequent per-row writes that key on that `block-id` (for example, `(Qo.Page.Bl/page-block ...)` reads or `(Qo.Public.OqlApi.Page.Bl/write! ...)` writes) collide on the page-block primary key and the query fails with `KEY_UPDATE_CONFLICT`.

Observed in MAB Strategist (2026-04-19). `add-or-get-by-name!` was invoked across 3 row pages where **one** of them already had a block with the target name from an earlier run. All 3 invocations bound `Block` to that one pre-existing block (`0d6fc01a-7ed4-4f27-9c2e-b24d9c9ba484`). Writing 3 different `RowPageId` values against one shared `BlockId` violated the page-block primary key.

## Why

The clause body (`query-omega-api/oql/app/public/oql-api/page/block/html.oql`) uses a `when-not` guard to decide whether to create a new block:

```oql
(:- (Qo.Public.OqlApi.Page.Bl.Html/add-or-get-by-name! Page Name Block)
    (get Page "page-id" PageId)
    (get Page "app-id" AppId)
    (when-not (and (Qo.Page.Bl/page-block _ PageId _ Block)
                   (get Block "type" "omega/query-omega/page/block/html/HtmlPageBlock")
                   (get Block "name" Name))
      (and (uuid BlockId)
           (= NewBlock { ... "block-id" BlockId ...})
           (Qo.Page.Bl/append-last Page NewBlock Block))))
```

`when-not` is a **universal quantifier over the current solution set**. The body fires only if the guard condition fails for **every** solution row. If the guard succeeds for even one row — because one `PageId` in the set happens to have a pre-existing block with the matching type and name — the body is skipped for **all** rows. `Block` is then bound, via the guard's `(Qo.Page.Bl/page-block _ PageId _ Block)` read, to the pre-existing block — and that single binding is shared across every row.

This compounds the `uuid` sharing described in [`call` vs `run-term` solution scoping](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/explanation/call-vs-run-term-on-captured-helpers.md) (per-row freshness section): even when the body does fire, a single-solution invocation would give every row the same `BlockId`. `when-not` adds a second failure mode on top — a guard that succeeds for one row suppresses creation for all rows.

Both failure modes have the same root cause: clause invocations that span multiple rows execute once against the full solution set, not N times against N single-row solutions.

## Fix

Wrap the per-row work in a stored helper clause and invoke it via `run-term` inside the iteration. `run-term` forces a fresh solution per invocation, so the `when-not` guard is re-evaluated against a single-row solution and `uuid` fires fresh every time.

```oql
;; BAD — one solution spans all rows.
;; If any row has a pre-existing match, when-not skips creation for all rows
;; and Block is bound to the pre-existing block for every row.
(Qo.Public.OqlApi.Page.Bl.Html/add-or-get-by-name! RowPage Name Block)
(Qo.Public.OqlApi.Page.Bl/write! RowPage Block _)
;; On 3 rows where one already has the block: all 3 writes use that one
;; shared Block → KEY_UPDATE_CONFLICT.
```

```oql
;; GOOD — helper clause invoked via run-term forces a fresh solution per row.
;; The when-not guard and uuid both evaluate independently for each row.
(:- (MyNs/ensure-html-block! RowPage Name _)
    (Qo.Public.OqlApi.Page.Bl.Html/add-or-get-by-name! RowPage Name Block)
    (Qo.Public.OqlApi.Page.Bl/write! RowPage Block _))

(run-term MyNs/ensure-html-block! RowPage Name _)
```

## Generalization

This is not specific to `add-or-get-by-name!`. Any public clause whose body uses `when` or `when-not` as a create-if-missing guard has the same multi-row hazard: a guard that succeeds for one row in the set suppresses (or triggers) the body for all rows, and any symbol bound inside the guard is shared across every row.

If you are calling a public clause that performs a create-if-missing pattern across multiple solution rows, wrap it in a `run-term` helper by default. If the clause only reads or is idempotent on a single shared binding, direct invocation across rows is fine.

## Related

- [`call` vs `run-term` solution scoping](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/explanation/call-vs-run-term-on-captured-helpers.md) — the underlying clause-boundary semantics and why `run-term` is the fix for per-row identity (per-row freshness section). Also covers the partition-before-fold pattern and a Correction section retracting an earlier misdiagnosed "captured-helper dispatch" rule. Read this first for the general principle; this doc adds the `when`/`when-not` universal-quantifier dimension on top.
- [run-term does not work on in-memory clauses](run-term-in-memory-clause.md) — `run-term` requires a stored functor. The helper clause must be stored.
- [OQL Execution Model — when / when-not](../explanation/oql-execution-model.md#when--when-not) — control-flow semantics of the conditional guards.
- [OQL Execution Model — Clauses as Solution Boundaries](../explanation/oql-execution-model.md#clauses-as-solution-boundaries) — why clause invocations share a solution and what that implies for bindings.
