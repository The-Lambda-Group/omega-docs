[< Home](../README.md)

# Gotchas

> **Confidentiality: PUBLIC.** All content in this directory is public platform documentation.

Failure modes, footguns, and debugging patterns. Read these when something surprising happened or when you are trying to diagnose a bug. These are "why did X fail" docs -- if you need design-level rationale, see [explanation/](../explanation/README.md); if you need exact syntax, see [reference/](../reference/README.md).

## Information ownership

This bucket owns gotchas that are part of the **public OQL language surface** -- scoping rules, boolean handling, return conventions, index matching. Deeper project-specific OQL gotchas (e.g., internal datastore quirks, pipeline-specific failure modes) live in `omega-knowledge-base/projects/query-omega-oql/gotchas/`.

## Navigation map

### Language-level

- [Lexical scope](lexical-scope.md) -- How scoping works in OQL clause bodies, and the subtle bugs that arise from getting it wrong. Covers the push-time vs runtime scope split (top-level symbols are not available inside clause bodies), what survives into a clause body (only head symbols and namespaced datastore references in namespace position -- everything else is renamed to gensyms), datastore behavior in clause bodies (namespace syntax works, value-position does not), why functors do not work inside stored clause bodies (use `run-page` instead), capture only working with `call`, the `_` wildcard, and the five most common scoping mistakes (using top-level variables in clauses, forgetting datastore redeclaration, expecting functor persistence, binding extra index variables, using `println-stream` before `Result`). Keywords: lexical scope, push time, runtime, clause body, symbol renaming, gensym, gen-local-symbols, datastore, namespace, functor, capture, call, underscore, stale value.

- [Conventions](conventions.md) -- Boolean stringification, return-binding rules, and common footguns that cause silent failures or crashes. Covers the boolean crash (OQL does not support boolean `false` at runtime -- always use strings `"true"`/`"false"`), return value rules (`return` must take an array, map literals inside `return` produce unresolved symbolic values -- bind first), the public API convention (component code must use `Qo.Public.*` datastores, never internal ones), index matching rules (bind exactly the fields in the index, no extra bound vars, no partial matches, row-index keys are always arrays), and datastore/functor gotchas inside stored clauses (redeclare datastores inside clause bodies, functors do not work inside stored clause bodies). Keywords: boolean, false, crash, return, bind before return, Qo.Public, internal datastore, index matching, row-index, array key, datastore redeclaration, functor, stored clause.

### Runtime limitations

- [call inside with-table-if scales poorly](call-inside-with-table-if.md) -- Calling an in-memory clause via `call` inside a `with-table-if` branch works for small inputs but hangs at ~15+ calls. Conjunction nesting from the call term solution fix creates exponentially deeper solution structures. Workaround: pre-compute outside the branch, use datastore-level aggregation, or limit batch size. Keywords: call, with-table-if, hang, scale, conjunction nesting, in-memory clause, performance.

- [run-term does not work on in-memory clauses](run-term-in-memory-clause.md) -- `run-term` expects a functor object from a datastore. Using it on an in-memory clause (bound in the solution table) produces a 500 error or, in capture-routed shapes, an engine spin-out / `ECONNRESET` requiring a database restart. **The rule for in-memory captured helpers: use `call` by default; `run-term` is appropriate ONLY when replacing `with-group-by` (the partition-before-fold scaling pattern). In any other context — one-solution-in / one-solution-out helpers, list-mapping helpers, validation helpers — `run-term` is wrong.** Working impls in the codebase that use `run-term` on captured helpers are doing partition replacement; do not pattern-match on those uses without confirming the partition framing. Keywords: run-term, call, in-memory clause, captured helper, functor, 500 error, ECONNRESET, engine spin-out, with-group-by, partition-before-fold, default rule.

- The `run-term` vs `call` rule (per-row freshness, partition-before-fold) has been consolidated into [`query-omega-oql/docs/explanation/call-vs-run-term-on-captured-helpers.md`](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/explanation/call-vs-run-term-on-captured-helpers.md). That doc covers the per-row UUID / `KEY_UPDATE_CONFLICT` symptom (was `run-term-vs-call-per-row.md`), the unified "when to use which" table, and a Correction section retracting an earlier misdiagnosed "captured-helper dispatch" rule that was disproved by bisection on 2026-04-26.

- [add-or-get-by-name binds the same block across multiple solution rows](add-or-get-by-name-multi-row.md) -- `Qo.Public.OqlApi.Page.Bl.Html/add-or-get-by-name!` uses a `when-not` guard to decide whether to create a new block. `when-not` is a universal quantifier over the current solution set: if the guard succeeds for any row (e.g. one page already has a matching block), the body is skipped for every row and `Block` is bound to the pre-existing block for all rows -- producing shared `block-id` writes and `KEY_UPDATE_CONFLICT`. Generalizes to any public clause whose body is a `when`/`when-not` create-if-missing guard. Fix: wrap in a helper clause invoked via `run-term` to force fresh per-row guard evaluation. Keywords: when-not, when, universal quantifier, add-or-get-by-name, create-if-missing, multi-row, solution set, shared binding, KEY_UPDATE_CONFLICT, run-term.

- [with-table-if header must include captured clause symbols](with-table-if-capture-header.md) -- When calling a captured in-memory clause inside a `with-table-if` branch, the clause symbol must be listed in the header array. Without it, the symbol is unbound in the branch scope. Keywords: with-table-if, header, capture, clause symbol, scope, unbound.
