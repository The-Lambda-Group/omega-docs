[< Home](../README.md) | [Reference](README.md)

# Clauses

A clause defines a rule — when its head is true based on its body. Clauses are defined with `(:- ...)`:

```oql
(:- (Head Arg1 Arg2 Result)
    (some-term Arg1 Intermediate)
    (other-term Intermediate Result))
```

Read this as: **Head is true if the body terms are all true.**

## Clause size and decomposition

A clause body is a unit of debugging. When something throws inside a body, the stack trace dumps the entire body — the larger the body, the harder it is to localize the failure. There are also second-order effects: long bodies tend to grow nested `with-table-if`s, which compound the same problem (see [Control flow → Nesting](control-flow.md#nesting)).

The discipline:

1. **5–10 lines per clause body.** If a body grows past ~10 terms, split it into smaller helper clauses. There is no engine-enforced limit — this is a debugging discipline, not a parser rule. Long bodies still run; they just become much harder to triage when something goes wrong, especially under high-fan-out load (LLM calls, large folds) where the failure is some terms deep into the body.
2. **No nested `with-table-if`s in any clause body.** Use **flat sequential** `with-table-if`s instead — each one does one piece of classification, the next reads the result via normal per-row unification. This is documented in detail at [Control flow → Nesting](control-flow.md#nesting); the rule is repeated here because it is the dominant reason a clause body grows past 10 lines.
3. **Compose via `(call SmallHelper ...)`, not via inline `(and ...)` blocks.** A 90-line `(and ...)` holding 15 distinct concerns is the anti-pattern. Stack traces from inside the `(and ...)` lose the branch boundary — you cannot tell from the trace which of the 15 concerns was executing. Lift each concern into its own named helper clause, and the trace tells you which helper was active. The named symbol *is* the breadcrumb.
4. **Each helper that calls captured helpers must declare its own `{"capture" [...]}`.** Captures do not propagate through nested `call` / `run-term` — a helper invoked via `call` from inside another clause does not inherit the outer clause's capture set. Each level that needs a captured symbol must list it. See [Closures / Capture](#closures--capture) below.

### Decomposition shape

The canonical shape for a workflow OnEvent or other large entry-point clause is:

- **One `BuildCtx`/`ResolveCtx` helper** that resolves the page hierarchy and database ids into a single `Ctx` map. Threads downstream as a single schema element instead of N separate ids — see [Control flow → The Ctx-threading pattern](control-flow.md#the-ctx-threading-pattern).
- **A handful of read helpers** (one per source — read filter inputs, read memory, read reports) each 5–10 lines, each pulling its own fields out of `Ctx`.
- **One or more compute helpers** (batch, build user message, dispatch LLM, etc.) each 5–10 lines.
- **Write helpers** (one per destination), each invoked via `(call WriteHelper ...)` from a parent clause.
- **A thin entry-point clause** (OnEvent, Run, Install, etc.) — usually 5–15 lines — that captures the top-level helpers and orchestrates the sequence of `(call ...)` invocations. The entry-point body is mostly pipe and gating, not logic.

### Working exemplar

The MAB Strategist post-refactor implementation is the canonical cargo-cult shape: 19 helpers each 5–10 lines, OnEvent body 11 lines (`~/Development/wttc/mab/impl/a24d83d6-workflows/83f5d9b7-design/b07b8100-mab-strategist/e9ca2103-implementation.oql`, commit `b779ca8`). Sketch:

```oql
;; Resolves page hierarchy and db-ids into a single Ctx map.
(:- (ResolveStratCtx Exec Ctx)
    (get-in Exec ["method" "page-id"] SelfPageId)
    (Qo.Public.OqlApi.Page/page-by-id SelfPageId SelfPage)
    ;; ...parent-walk + child-page-by-name...
    (= Ctx {"brief-page" BriefPage
            "instructions-db-id" InstructionsDbId
            "treatments-db-id" TreatmentsDbId
            ;; ...
            }))

;; Reads filter inputs as flat sequential with-table-ifs (no nesting).
(:- (ReadStratFilter Ctx ActiveCount WrittenCount UnwrittenIds UnwrittenCount
                     {"capture" [UnwrittenTreatments]})
    (get Ctx "treatments-db-id" TreatmentsDbId)
    (get Ctx "strategies-db-id" StrategiesDbId)
    (with-table-if (Qo.Public.OqlApi.Db.Prop/prop-vals-by-folder TreatmentsDbId AnyTPid _)
      [TreatmentsDbId AnyTPid ActiveIds]
      ;; ...then-branch...
      (= ActiveIds []))
    (with-table-if (Qo.Public.OqlApi.Db.Prop/prop-vals-by-folder StrategiesDbId AnySPid _)
      [StrategiesDbId AnySPid WrittenIds]
      ;; ...then-branch...
      (= WrittenIds []))
    (call UnwrittenTreatments ActiveIds WrittenIds UnwrittenIds)
    (length ActiveIds ActiveCount)
    (length WrittenIds WrittenCount)
    (length UnwrittenIds UnwrittenCount))

;; Thin entry point — captures the helpers and orchestrates the sequence.
(:- (OnEvent Exec Result
             {"capture" [ResolveStratCtx ReadStratFilter ProcessStratBatch LatestInstrContent]})
    (call ResolveStratCtx Exec Ctx)
    (get Ctx "brief-page" BriefPage)
    (get Ctx "instructions-db-id" InstructionsDbId)
    (Qo.Public.OqlApi.Page.Bl.Html/get-content BriefPage "Content" BriefContent)
    (call LatestInstrContent InstructionsDbId "strategist" InstrContent)
    (call ReadStratFilter Ctx ActiveCount WrittenCount UnwrittenIds UnwrittenCount)
    (with-table-if (> UnwrittenCount 0)
      [Exec Ctx UnwrittenIds UnwrittenCount BriefContent InstrContent Result ProcessStratBatch]
      (call ProcessStratBatch Exec Ctx UnwrittenIds UnwrittenCount BriefContent InstrContent Result)
      (= Result {"has-more" "false" "unwritten-count" 0 "skipped" "no-work"})))
```

The MAB Enumerate implementation shows the same shape with a `BuildCtx` + thin OnEvent (`~/Development/wttc/mab/impl/a24d83d6-workflows/83f5d9b7-design/d230922b-mab-enumerate/02a9dbd9-implementation.oql`, lines 130–185).

### Anti-pattern (what NOT to do)

The pre-refactor Strategist OnEvent (Strategist commit `c681381`, 150 lines, with a 90-line nested `(and ...)` block holding 15 distinct concerns) is the shape to avoid. It executed correctly, but during cold-start `EOFException` debugging the stack trace pointed at "somewhere inside the `(and ...)`" — there was no way to tell which of the 15 concerns had thrown. The refactor to the shape above made the same failure trivially localizable: the trace named the helper.

The discipline is project lore distilled from that experience. Apply it from the start of every workflow implementation, not as a post-hoc cleanup.

## Stored Clauses (written to the database)

When a clause head is namespaced with a datastore and ends with `!`, it's **stored** in that datastore. It persists across queries — any future query can call it.

```oql
(datastore Qo.Page "omega/query-omega/page")

;; This clause is STORED in the Qo.Page datastore
(:- (Qo.Page/visible! PageId)
    (Qo.Page/page-object _ ParentId PageId _)
    (Qo.Page.Bl/page-block _ ParentId _ Block)
    (get Block "page-id" PageId))
```

The `!` suffix tells the engine to write the clause to the datastore. Without `!`, calling the clause reads from the datastore. This is the read/write convention:

```oql
(Qo.Page/visible PageId)   ;; READ — query the stored clause
(Qo.Page/visible! PageId)  ;; WRITE — define/store the clause
```

## In-Memory Clauses (bound to a symbol)

When a clause head has no namespace prefix and no `!`, it's **not stored** anywhere. It exists only as a symbol binding in the current query. Think of it like a local function.

```oql
;; This clause is NOT stored — it's just bound to the symbol ParseSales
(:- (ParseSales RawSales Sales)
    (with-table-if (= RawSales "")
      [RawSales Sales]
      (= Sales 0)
      (json-stringify Sales RawSales)))

;; You can call it in the same query using `call`
(call ParseSales "100" Result)   ;; Result = 100
```

In-memory clauses are used for:
- **Helper logic** inside an implementation (parsing, scoring, transforming)
- **Implementation clauses** that get registered with `set-implementation-clauses`
- **Captured clauses** passed into stored clauses via the `capture` mechanism

## `clause` Term Syntax

The `clause` term is an alternative way to define an in-memory clause. Instead of `(:- (Name Args...) body...)`, you write:

```oql
(clause Name [Arg1 Arg2]
    (some-term Arg1 Intermediate)
    (other-term Intermediate Arg2))
```

This creates a `ClauseTerm` that binds `Name` as a callable clause in the current solution. Call it with `call`:

```oql
(clause ParseValue [Raw Parsed]
    (with-table-if (= Raw "")
      [Raw Parsed]
      (= Parsed 0)
      (json-stringify Parsed Raw)))

(call ParseValue "100" Result)   ;; Result = 100
```

The key difference from `(:- ...)`: the `clause` form takes arguments as a vector `[Arg1 Arg2]` rather than as part of the head `(Name Arg1 Arg2)`. Both forms produce in-memory clauses invoked with `call`.

## Inner Clauses (experimental)

You can define an in-memory clause inside another clause's body. It's scoped to that execution.

```oql
(:- (MyOuterClause Data Result)
    ;; Define a helper clause
    (:- (ParseValue Raw Parsed)
        (with-table-if (= Raw "")
          [Raw Parsed]
          (= Parsed 0)
          (json-stringify Parsed Raw)))

    ;; Use it
    (get Data "value" RawVal)
    (call ParseValue RawVal Parsed)
    (= Result {"parsed" Parsed}))
```

**Warning:** Inner clauses are highly experimental. Known limitations:
- **Stored clause writes inside inner clauses don't persist.** A `(:- (Ds/term! ...))` defined inside another clause body will not write to CouchDB. The stored clause write only works at the top level of a query. If you need to define a stored clause dynamically, define it at the top level and use `capture` to pass it into the clause that needs it.
- Inner in-memory clauses (no `!`) work for helper logic within a single query execution.

## Closures / Capture

Clauses can capture symbols from the outer scope using the `capture` syntax:

```oql
(:- (MyClause A B Result {"capture" [SomeOuterVar]})
    ;; SomeOuterVar is available here from the outer scope
    (+ A SomeOuterVar Result))
```

This is the recommended workaround when you need a stored clause's result available inside an implementation clause. Define the stored clause at the top level (push time), create a functor for it, and capture the functor:

```oql
;; At push time — define the stored clause and create a functor
(datastore HandlerDs "my/handler/datastore")

(:- (HandlerDs/handle-event! Event Result)
    ;; ... handler logic ...
    (= Result {"status" "ok"}))

(functor "handle-event" 2 HandlerDs HandleFunc)

;; Implementation clause captures the functor from push time
(:- (Install Exec Result {"capture" [HandleFunc]})
    ;; HandleFunc is available here — pass it to write-event-subscription, etc.
    (= Result {"handler" HandleFunc}))
```

**Warning:** Capture is experimental. There are known edge cases with capturing clauses inside other captured clauses. Keep capture usage flat — avoid nesting captured clauses.

### Related gotchas

The capture mechanism interacts with several scoping and invocation rules documented in the gotchas tree. If you are diagnosing a runtime error that may originate in capture semantics, scan these:

- [Lexical scope](../gotchas/lexical-scope.md) — what survives into a clause body, the push-time vs runtime scope split, and the "Capture Only Works With call" rule (a captured clause symbol must be invoked with `call`, not as a bare functor reference).
- [with-table-if header must include captured clause symbols](../gotchas/with-table-if-capture-header.md) — the per-branch scope-boundary corollary: even after capture has threaded a clause symbol into the surrounding clause body, a `with-table-if` branch needs that symbol listed in its header to see it.
- [add-or-get-by-name binds the same block across multiple solution rows](../gotchas/add-or-get-by-name-multi-row.md) — when the fix is "wrap in a stored helper invoked via `run-term`," capture is how you hand the helper the bindings it needs from the surrounding scope.
- [run-term does not work on in-memory clauses](../gotchas/run-term-in-memory-clause.md) — capture targets must be stored functors when reached via `run-term`. Define the stored clause at the top level, build a functor, and capture the functor. **Default rule for in-memory captured helpers: use `call`. `run-term` against a captured in-memory helper is appropriate ONLY when replacing `with-group-by` (partition-before-fold scaling); in any other context it produces a 500 or engine spin-out / `ECONNRESET`.**
