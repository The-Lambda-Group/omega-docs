[< Home](../README.md) | [OmegaDB](../README.md#omegadb)

# OQL — Omega Query Language

OQL is a logic programming language inspired by Datalog and Prolog. Instead of writing step-by-step procedures, you describe relationships and constraints, and the database engine finds all solutions that satisfy them.

## Terms

The basic unit of OQL is a **term** — a statement that is either true or false.

```oql
(blue "Sky")           ;; "Sky" is blue
(even 2)               ;; 2 is even
(- 6 1 5)              ;; 6 - 1 = 5
```

Terms are evaluated by the database. If a term is true (there's a matching fact or rule), it succeeds. If false, it fails.

## Symbols and Unification

Capitalized names are **symbols** (variables). The engine tries to find values for symbols that make all terms true simultaneously.

```oql
(= X 5)               ;; X is bound to 5
(+ X 1 Y)             ;; Y is bound to 6 (because X is 5)
```

This is **unification** — the engine doesn't assign variables top-down like imperative code. It finds consistent bindings across all terms.

## Datastores

Data in OQL lives in **datastores**. A datastore is a named location that holds terms (facts and rules). You declare a datastore and then use it as a namespace prefix:

```oql
(datastore Qo.Page "omega/query-omega/page")

;; Query the page datastore
(Qo.Page/page-object _ _ PageId Page)
```

The string `"omega/query-omega/page"` is the datastore's path in the database. The symbol `Qo.Page` is how you refer to it in the query.

## Clauses

A clause defines a rule — when its head is true based on its body. Clauses are defined with `(:- ...)`:

```oql
(:- (Head Arg1 Arg2 Result)
    (some-term Arg1 Intermediate)
    (other-term Intermediate Result))
```

Read this as: **Head is true if the body terms are all true.**

### Stored Clauses (written to the database)

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

### In-Memory Clauses (bound to a symbol)

When a clause head has no namespace prefix and no `!`, it's **not stored** anywhere. It exists only as a symbol binding in the current query. Think of it like a local function.

```oql
;; This clause is NOT stored — it's just bound to the symbol ParseSales
(:- (ParseSales RawSales Sales)
    (with-table-if (= RawSales "")
      [RawSales Sales]
      (= Sales 0)
      (json-stringify Sales RawSales)))

;; You can call it in the same query
(ParseSales "100" Result)   ;; Result = 100
```

In-memory clauses are used for:
- **Helper logic** inside an implementation (parsing, scoring, transforming)
- **Implementation clauses** that get registered with `set-implementation-clauses`
- **Captured clauses** passed into stored clauses via the `capture` mechanism

### Inner Clauses (experimental)

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
    (ParseValue RawVal Parsed)
    (= Result {"parsed" Parsed}))
```

**Warning:** Inner clauses are highly experimental. There are known bugs with clause definitions inside clause bodies. Test thoroughly and expect edge cases.

### Closures (experimental)

Clauses can capture symbols from the outer scope using the `capture` syntax:

```oql
(:- (MyClause A B Result {"capture" [SomeOuterVar]})
    ;; SomeOuterVar is available here from the outer scope
    (+ A SomeOuterVar Result))
```

**Warning:** Closures are also highly experimental. There are several known bugs with capturing clauses inside clauses. Use with caution.

## Functors

A **functor** is a named function signature with an arity (argument count) bound to a datastore. Functors are how the engine knows where to find clauses.

```oql
;; Create a functor: name "handle-event", arity 2, in HandlerDs datastore
(functor "handle-event" 2 HandlerDs HandlerFunc)
```

The resulting `HandlerFunc` is an object that points to clauses stored under the name "handle-event" with arity 2 in the `HandlerDs` datastore. You can invoke it with `call`:

```oql
;; Invoke the functor — finds matching clauses and executes them
(call HandlerFunc Event Result)
```

When you write `(Qo.Page/page-object _ _ PageId Page)`, this is syntactic sugar — the engine resolves `Qo.Page` to a datastore, creates a functor for `page-object` with arity 4, and calls it.

## defterm

`defterm` declares a term schema on a datastore — it defines what terms look like and which arguments are the primary key. This creates the necessary CouchDB indexes for efficient lookups.

```oql
(datastore Qo.Data.LogStream "omega/query-omega/data/log-stream")

(defterm Qo.Data.LogStream
  {"name" "log-stream-data"
   "args" ["AppId" "LogStreamId" "LogStream"]
   "primary-key" ["LogStreamId"]})
```

This says: the `Qo.Data.LogStream` datastore holds terms named `log-stream-data` with 3 arguments, and `LogStreamId` (the second argument) is the primary key.

After defining a defterm, you can read and write terms:

```oql
;; WRITE — store a term (! suffix)
(Qo.Data.LogStream/log-stream-data! AppId LogStreamId LogStream)

;; READ — query for terms
(Qo.Data.LogStream/log-stream-data AppId LogStreamId LogStream)

;; DELETE
(Qo.Data.LogStream/delete! (log-stream-data AppId LogStreamId _))
```

## write-index

Creates a secondary index on a datastore for faster lookups by specific argument positions.

```oql
(datastore MyDs "my/datastore/path")
(functor "my-term" 3 MyDs MyFunc)

;; Create an index on the first argument (args.0)
(write-index MyFunc "my/datastore/path/MyIndex" ["args.0"])

;; Create a compound index on first and second arguments
(write-index MyFunc "my/datastore/path/CompoundIndex" ["args.0" "args.1"])
```

The index path is a unique identifier for the index. The array specifies which argument positions to index. Queries that bind exactly those positions will use the index instead of scanning the full datastore.

## Maps and Access

OQL works natively with JSON-like maps:

```oql
;; Read a key
(get Page "name" Name)

;; Read a nested key
(get-in Page ["page-data" "name"] Name)

;; Set a key (returns new map)
(set Page "status" "active" UpdatedPage)

;; Create a map
(= Config {"api-key" "secret" "endpoint" "https://api.example.com"})
```

## Control Flow

OQL's control flow operates on solution sets. Understanding the distinction between universal quantifiers (`when`, `when-not`, `if`) and per-solution branching (`with-table-if`) is critical.

### when / when-not / if (universal quantifiers)

These operate as **"if any"** — they pass or fail based on whether *any* solution in the current solution set satisfies the condition. They don't branch per-solution.

```oql
;; (when) — "when any": runs body if the condition succeeds for any solution
(when (= Step "score")
  (= PageName "Score Agents"))

;; (when-not) — "when no": runs body if the condition fails for all solutions
(when-not (get Data "skip" Skip)
  (= Skip 0))

;; (if) — "if any": binds one of two values based on whether condition succeeds
(if (get Data "continue" "false")
  (= ShouldContinue "false")
  (= ShouldContinue "true"))
```

Use `when` for flat dispatch (like a switch/case):
```oql
(when (= Step "score")  (= PageName "Score Agents"))
(when (= Step "filter") (= PageName "Filter Agents"))
(when (= Step "batch")  (= PageName "GHL Sync"))
```

### with-table-if (per-solution branching)

`with-table-if` is fundamentally different — it branches **per solution**. The condition evaluates within the parent solution, and each branch creates its own table solution.

```oql
(with-table-if Condition
  [Var1 Var2 Result]          ;; table head — variables in scope for both branches
  ThenBranch                   ;; runs for solutions where Condition succeeds
  ElseBranch)                  ;; runs for solutions where Condition fails
```

The table head `[Var1 Var2 Result]` declares which symbols flow into both branches. This is how you **conditionally bind variables**:

```oql
;; Not every Data has "key" — conditionally bind Val
(with-table-if (get Data "key" Val)
  [Data Val Result]
  (= Result {"found" Val})           ;; then: Val is bound
  (= Result {"found" "default"}))    ;; else: Val was not in Data
```

Inside each branch, you can define new symbols with `(and ...)` — they get wrapped into the table:

```oql
(with-table-if (= RawSales "")
  [RawSales Sales]
  (= Sales 0)                        ;; then: empty string → 0
  (and (string-replace RawSales "," "" Clean)    ;; else: parse it
       (json-stringify Sales Clean)))             ;; new symbols OK inside (and)
```

Both branches evaluate and the results are combined — essentially an `(or)` of the two tables. The solution set continues with all rows from both branches.

**Nesting** is allowed for multi-step parsing but prefer flat `when` blocks or separate clauses when possible:

```oql
;; Nested with-table-if for parsing (acceptable pattern)
(with-table-if (= RawSales "")
  [RawSales Sales]
  (= Sales 0)
  (with-table-if (number RawSales)
    [RawSales Sales]
    (= Sales RawSales)
    (and (string-replace RawSales "," "" Clean)
         (json-stringify Sales Clean))))
```

### When to use which

| Pattern | Use when |
|---------|----------|
| `when` | Flat dispatch — run body if condition matches any solution |
| `when-not` | Default values — bind a symbol when it's not already bound |
| `if` | Bind one of two values based on a condition |
| `with-table-if` | Per-solution branching — conditionally bind variables, parse values |

## Built-in Terms

Common built-in operations:

```oql
(= X 5)                              ;; unify X with 5
(+ 1 2 Sum)                          ;; arithmetic
(get Map "key" Value)                 ;; map access
(get-in Map ["a" "b"] Value)          ;; nested access
(set Map "key" NewVal Result)         ;; map update
(string-split Path "/" Segments)      ;; string operations
(uuid Id)                             ;; generate UUID
(json-stringify Obj Str)              ;; JSON serialization
(defined X)                           ;; check if symbol is bound
(return [Result])                     ;; return from a query
```

## Important Conventions

**Booleans:** OQL does not support boolean `false` at runtime — it will crash the engine. Always use strings:
```oql
;; GOOD
(= Result {"has-more" "true"})

;; BAD — crashes
(= Result {"has-more" false})
```

**Return values** must always be arrays:
```oql
(return [Result])
```

**Datastore access:** Component code should only use public API endpoints (`Qo.Public.*`), never internal datastores. If a public endpoint doesn't exist for what you need, create one in the API layer.
