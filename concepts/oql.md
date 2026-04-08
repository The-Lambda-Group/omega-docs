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

## Rules (Clauses)

A rule defines when a term is true based on other terms. In OQL, rules are defined with `(:- ...)`:

```oql
;; "A page is visible if it has a block on its parent"
(:- (Qo.Page/visible! PageId)
    (Qo.Page/page-object _ ParentId PageId _)
    (Qo.Page.Bl/page-block _ ParentId _ Block)
    (get Block "page-id" PageId))
```

Read `(:- (Head) (Body...))` as: **Head is true if Body is true.**

The `!` suffix is a convention meaning the clause writes to the datastore when defined. Without `!`, the clause only exists in the current query.

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

OQL doesn't have if/else in the traditional sense. Instead:

```oql
;; (when) — runs body if condition is true, no-op if false
(when (= Step "score")
  (= PageName "Score Agents"))

;; (when-not) — runs body if condition is false
(when-not (defined Var)
  (= Var "default"))

;; (with-table-if) — conditional with explicit variable scope
(with-table-if (get Data "key" Val)
  [Data Val Result]                    ;; variables in scope
  (= Result "found")                   ;; then branch
  (= Result "not found"))             ;; else branch
```

Use `(when)` for flat dispatch. Use `(with-table-if)` only when you need to branch on a condition and bind different values.

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
