[< Home](../README.md) | [How-to](README.md)

# Develop OQL implementations incrementally

OQL development is REPL-driven. You build queries incrementally, one step at a time, verifying each step before adding the next. The push-run cycle is fast enough to test every single change — there is no reason to batch work.

## Prerequisites

Before writing any OQL implementation, read:

- [OQL hard rules](../reference/oql-hard-rules.md) — non-negotiable rules on forbidden datastores, forbidden operations, and the boolean-string requirement. Read this first.
- [Execution model](../explanation/oql-execution-model.md) — how OQL evaluates queries (solution sets, why terms can't be nested inside literals)
- [Built-in terms](../reference/built-ins.md) — the full built-in term reference
- [Control flow](../reference/control-flow.md) — read this before writing any `when`, `when-not`, `if`, or `with-table-if`

Skipping these leads to violating the hard rules, using non-existent terms, misusing `if` (universal, not per-row), and missing idiomatic patterns like using `(get Map "key" Val)` as a condition in `with-table-if` (the `get` itself fails on null/missing, so the else branch provides the default).

## Required file structure: `set-implementation-clauses`

Every runnable OQL implementation file has a mandatory four-layer structure. **A bare OQL body (`(= Result "123") (return [Result])`) pushes successfully but fails at runtime** with `Qo.Data.Impl.Clauses/impl-clauses` no-return when called via `qo run`. The reason: `qo push` stores the OQL text, but `qo run` dispatches via the *compiled-clauses store* — a separate registry that a bare body never populates.

The four layers:

1. **Datastore declarations** — one `(datastore ...)` per namespace referenced anywhere in the file.
2. **In-memory clause definitions** — one `(:- (Method Exec Result) ...)` per protocol method.
3. **Clause registration** — build a Clauses map and call `set-implementation-clauses` to persist it.
4. **File-level return** — `(return ["ok"])` as the last line, required for `qo push` to succeed.

Minimum working example (Query protocol, one `run` method):

```oql
(datastore Qo.Public.OqlApi.Impl "omega/query-omega/public/oql-api/implementation")

(:- (Run Exec Result)
    (= Result "123"))

(= Clauses {"run" {"2" Run}})
(get $$ARG$$ "implementation-id" ImplId)
(Qo.Public.OqlApi.Impl/set-implementation-clauses ImplId Clauses _)
(return ["ok"])
```

The arity key `"2"` matches the 2-arg functor `(Run Exec Result)` — it must equal the protocol spec's `args` length as a JSON string.

> **See [write-implementation-impl-file.md](../../omega-ai-components/docs/how-to/write-implementation-impl-file.md) for the full explanation** — arity key derivation, multi-method Clauses maps, push-time vs dispatch-time scoping, and all common mistakes.

## The workflow

### 1. Start with the plumbing

Write the minimum query that proves the basic wiring works — page walk, push connector read, HTTP call. Return the raw result. Push, run, verify. Don't add anything else until this returns clean data.

```bash
qo push file.oql && qo run path -p Query -m run -a '[{}]'
```

### 2. Add one transformation at a time

Need to parse JSON? Add just `(json-stringify Parsed RawBody)` and return `Parsed`. Push, run, verify. Need to grab the first element? Add just `(get Parsed 0 First)` and return `First`. Push, run, verify. Never add two untested steps at once.

### 3. Never rewrite the file

When you get feedback on one thing, edit only that thing. Don't rewrite from scratch — you'll lose working code and reintroduce bugs you already fixed. Use incremental edits on the last known-working version.

### 4. Avoid side effects until logic is verified

Don't call `write-table` until you've returned the exact input you'd pass to it and confirmed the shape is correct. Return your would-be write-table args as the Result, inspect them, and only then swap in the actual write call. This is like dry-run mode.

```oql
;; Dry-run: inspect the write-table input before committing to the side effect
(= Result {"file-id" FolderId "data" {"header" Header "rows" Rows}})
;; After verifying the shape, swap in:
;; (Qo.Public.OqlApi.Db.Prop/write-table {"file-id" FolderId "data" {"header" Header "rows" Rows}} Result)
```

### 5. Debug by returning early

When something breaks, comment out everything after the suspected failure point and bind `Result` to the last intermediate value. This tells you exactly where the chain broke and what the data looks like at that point. See [Debug by returning early](debug-by-returning-early.md) for the full recipe.

### 6. One clause, one concern

If you need to stringify a field, get it, stringify it, set it back — three lines, return the result, verify. Don't combine stringify + write-table + count in one untested block.

## Logic tree balance

Two dimensions govern whether an OQL implementation stays manageable.

**Table header branching factor.** The number of solutions a `with-table-if` condition or clause head produces is the branching factor at that level. Keep it ≤6; even 6 requires justification. Prefer ≤4. To understand why the ceiling matters: a branching factor of 6 at three nested levels produces 6³ = 216 candidate solutions; at 4 it is 4³ = 64. Combinatorial blowup from unconstrained branching is the most common cause of logic trees that are correct in isolation but slow or undebuggable at full scale.

**Clause sub-clause count.** Each clause body should contain ≤4 sub-clauses or steps. Beyond four, the clause is doing too many things at once. Split it.

### Pre-seed map as a concrete example

The pre-seed map pattern keeps fold-body branching factor at 1. Before the fold begins, build a map that guarantees every expected key is present (using a default value for missing entries). Inside the fold body, `(get Map Key Val)` always resolves — it never widens the solution table because the key is always there. Compare this to a `with-table-if (get Map Key Val)` guard inside the fold body: if the map is missing keys, the condition fails for those rows and the else-branch fires, adding a branch. Seeding the map before the fold eliminates that branch entirely.

This also avoids the 9-entry map literal binding bug: if the map has ≥9 keys, build it in two sub-maps of ≤8 keys each and merge them before the fold. Each sub-map's `(=)` bind resolves correctly; `(merge MapA MapB Out)` combines them without going through the broken pathway.

### Cross-link

For binding rules inside `with-table-if` — which symbols must appear in the header, condition vs branch scope — see [Control flow](../reference/control-flow.md).

## What NOT to do

Don't write the entire query with HTTP call + JSON parse + field extraction + table write + error handling + count in one shot. It will fail somewhere in the middle and the 500 error won't tell you where. You'll waste time guessing and rewriting when you could have caught the issue in 30 seconds with an incremental test.

## The push-run cycle is fast

`qo push file.oql && qo run path -p Query -m run -a '[{}]'` takes seconds. There's no reason to batch work — the feedback loop is tight enough to test every single change.

## Bound symbols in `throw` data maps

> **Before writing any `throw` whose data map contains bound symbols, read [Built-in terms § Symbols in Literal Arguments](../reference/built-ins.md#symbols-in-literal-arguments).**

Symbol bindings do not resolve inside literal map or list arguments passed directly to any term — including `throw`. Writing the map inline causes the symbol name (e.g., `"FilterPropCheck17490"`) to be serialized into the thrown value instead of the symbol's runtime value (e.g., `"decision"`).

**Always bind the map to a variable first:**

```oql
;; WRONG — FilterPropCheck is serialized as the symbol name, not its value
(throw "FILTER_SORT_COMBINED_UNSUPPORTED" {"filter-prop" FilterPropCheck})

;; RIGHT — bind the map first, then throw the variable
(= ErrData {"filter-prop" FilterPropCheck})
(throw "FILTER_SORT_COMBINED_UNSUPPORTED" ErrData)
```

This applies to every term that accepts a map or list argument: `throw`, `run-page`, `set-data`, and all others. The rule and more examples are in [Built-in terms § Symbols in Literal Arguments](../reference/built-ins.md#symbols-in-literal-arguments).

## Variable names must not match datastore namespace final segments

> **Convention: never use a single-word variable name that matches the final segment of any `(datastore ...)` declaration in the same clause.**

The OQL gensym mechanism resolves variable names against the in-scope datastore bindings during recursive clause calls. If a variable name exactly matches the final segment of a datastore namespace path declared in the same clause, the engine can pre-bind the variable to the datastore reference object (a `clojure.lang.Symbol`) instead of leaving it free for the calling site to supply.

**Example collision:**

```oql
(datastore Qo.Data.Dl.Func "omega/query-omega/data/dl/func")

(:- (DeleteRow Exec Result)
    ;; WRONG — "Func" matches the final segment of Qo.Data.Dl.Func.
    ;; During recursive calls, Func is pre-bound to the datastore reference
    ;; (a clojure.lang.Symbol) instead of the caller's intended value.
    (= Func (get Exec "func-id"))
    ...)
```

**Correct approach — use a name that does not appear as any declared datastore's final segment:**

```oql
(datastore Qo.Data.Dl.Func "omega/query-omega/data/dl/func")

(:- (DeleteRow Exec Result)
    ;; RIGHT — "DeleteFunc" does not collide with any declared datastore.
    (= DeleteFunc (get Exec "func-id"))
    ...)
```

**Symptoms of a collision:** the variable resolves to a `clojure.lang.Symbol` value (the datastore object) at the call site instead of the value the caller bound. The bug only surfaces during recursive clause calls, not when the clause body is evaluated in isolation, which makes it hard to diagnose without knowing the rule.

**Scope of the rule:** check every `(datastore Ns.Path.Segment ...)` declaration in the same clause. The at-risk names are the final dot-separated segments — `Func` for `Qo.Data.Dl.Func`, `Page` for `Qo.Data.Page`, `Block` for `Qo.Data.Page.Block`, and so on. Rename any local variable that matches one of these to something descriptive and distinct (e.g., `DeleteFunc`, `TargetPage`, `ContentBlock`).

## Paginated table-read pipelines: paginate first, look up second

> **Anti-pattern: fold-full-table-into-map followed by slice.** If you read all N rows into a map and then apply `BatchTakeN` to narrow to a page of K keys, all N rows are loaded into memory regardless of K. At scale (N=1000, K=3) this is a full-table scan on every firing.

The correct pattern: **apply pagination to the filtered set first, then do per-key lookup**.

### Wrong: full-table-scan-into-map

```oql
;; ReadExperimentsMap — loads ALL N rows regardless of limit
(:- (ReadExperimentsMap Exec ExperimentsMap)
    (get Exec "experiments-folder-id" FolderId)
    (prop-vals-by-folder FolderId _ Pv)
    (get-in Pv ["prop-vals" "id"] ExpId)
    (get-in Pv ["prop-vals" "status"] Status)
    (fold {} [ExpId Status] assoc ExperimentsMap))

(:- (OnEvent Exec Result)
    (call ResolveLimit Exec LimitStr)
    (call ReadExperimentsMap Exec ExperimentsMap)  ; N rows loaded
    (keys ExperimentsMap AllKeys)
    (call BatchTakeN AllKeys LimitStr PageKeys)    ; then narrow to K keys
    ...)
```

This loads every row in the table on every Coordinator firing. At N=1000 with `limit=3`, the engine processes 1000 rows to return 3.

### Right: paginate first, look up second

```oql
(:- (ReadExperimentIds Exec ExperimentIds)
    (get Exec "experiments-folder-id" FolderId)
    (prop-vals-by-folder FolderId _ Pv)
    (get-in Pv ["prop-vals" "id"] ExpId)
    (with-order-by [ExpId]          ; deterministic order for cursor stability
      (fold [] ExpId append ExperimentIds)))

(:- (LookUpExperiment FolderId ExpId Row)
    (prop-vals-by-folder FolderId Pid Pv)
    (get-in Pv ["prop-vals" "id"] ExpId)
    (= Row Pv))

(:- (OnEvent Exec Result)
    (call ResolveLimit Exec LimitStr)
    (call ReadExperimentIds Exec AllIds)
    (call BatchTakeN AllIds LimitStr PageIds)      ; paginate the id list first
    (run-term LookUpExperiment [FolderId PageId] PageId PageIds Row Rows)
    ...)                                           ; then look up only the K rows
```

This reads only the PK column on every firing. The per-row lookup (`LookUpExperiment`) runs only K times.

### When this matters

The overhead difference is small at development scale (N=5-10 rows). It becomes significant once the table grows beyond a few hundred rows. Establishing the paginate-first shape from the start costs nothing and prevents a later rewrite.

The pattern is the same one used by Execute and Sync Results (see [code-step-pagination-pattern.md](../explanation/code-step-pagination-pattern.md) for the full helper set: `ResolveSkip`, `ResolveLimit`, `BatchTakeN`, `ComputeHasMore`).

> **Source:** I7 in session 2026-05-10-smartlead-schedule-error-propagation. The anti-pattern was identified as pre-existing tech debt in `ReadExperimentsMap`, which folds the entire Experiments table into a map before `BatchTakeN` narrows to `limit` keys — N rows loaded, K rows processed.
