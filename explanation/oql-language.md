[< Home](../README.md) | [Explanation](README.md)

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

**Unification applies to every term, not just `=`.** When you write `(get Map "key" Value)`, the third argument is unified — if `Value` is an unbound symbol, it binds to whatever the key holds. If `Value` is a literal, the term only succeeds when the key's value matches that literal:

```oql
(= Data {"step" "score" "flag" "false"})

;; Symbol in third position — binds Val to "false"
(get Data "flag" Val)          ;; succeeds, Val = "false"

;; Literal in third position — unifies, so this is a constraint
(get Data "flag" "false")      ;; succeeds (value matches)
(get Data "flag" "true")       ;; FAILS (value is "false", not "true")

;; Key missing — term fails regardless
(get Data "missing" Val)       ;; FAILS (key doesn't exist)
```

This matters for control flow. `(if Condition Then Else)` checks whether `Condition` **succeeds as a term**, not whether it returns a truthy value. So `(if (get Data "flag" "false") ...)` asks "does the key `flag` exist with value `"false"`?" — not "get flag with default false":

```oql
(= Data {"step" "score"})

;; "flag" is missing — get fails — takes Else branch
(if (get Data "flag" "false")
  (= Result "then")
  (= Result "else"))           ;; Result = "else"

;; Now with flag present:
(= Data2 {"step" "score" "flag" "false"})

;; "flag" exists and value matches "false" — get succeeds — takes Then branch
(if (get Data2 "flag" "false")
  (= Result2 "then")           ;; Result2 = "then"
  (= Result2 "else"))
```

## Maps and Access

OQL works natively with JSON-like maps. Remember — every argument is unified, so `get` and `get-in` can both **read** (bind an unbound symbol) and **constrain** (match against a literal):

```oql
;; Bind Name to whatever value is at "name"
(get Page "name" Name)

;; Constrain — only succeeds if name is "Score Agents"
(get Page "name" "Score Agents")

;; Nested access (same rules apply)
(get-in Page ["page-data" "name"] Name)

;; Set a key (returns new map)
(set Page "status" "active" UpdatedPage)

;; Create a map
(= Config {"api-key" "secret" "endpoint" "https://api.example.com"})
```

## Sub-pages

- [Datastores, Functors & defterm](language/datastores.md) — data storage, functors, term schemas, secondary indexes
- [Clauses](language/clauses.md) — stored, in-memory, inner clauses, closures/capture
- [Control Flow](language/control-flow.md) — when, when-not, if, with-table-if
- [Built-in Terms](language/built-ins.md) — core, arithmetic, maps, arrays, strings
- [Reducers](language/reducers.md) — fold, with-unique-by, with-order-by, with-group-by, with-count, with-limit
- [Conventions & Gotchas](language/conventions.md) — booleans, return values, datastore access, index matching
