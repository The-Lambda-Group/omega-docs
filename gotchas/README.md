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

- [run-term does not work on in-memory clauses](run-term-in-memory-clause.md) -- `run-term` expects a functor object from a datastore. Using it on an in-memory clause (bound in the solution table) produces a 500 error. Use `call` instead. Keywords: run-term, call, in-memory clause, functor, 500 error.

- [with-table-if header must include captured clause symbols](with-table-if-capture-header.md) -- When calling a captured in-memory clause inside a `with-table-if` branch, the clause symbol must be listed in the header array. Without it, the symbol is unbound in the branch scope. Keywords: with-table-if, header, capture, clause symbol, scope, unbound.
