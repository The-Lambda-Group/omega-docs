# Lexical Scope in OQL

Understanding how scope works in OQL is critical for writing correct implementations. OQL has different scoping rules than most languages, and getting them wrong produces subtle bugs where clauses appear to work but silently use stale or wrong values.

## Push Time vs Runtime Scope

An implementation file has two execution contexts:

**Top-level scope** — runs once at push time. Symbols bound here are available to other top-level statements in the same file.

**Clause body scope** — defined at push time, runs later at runtime. Symbols inside `(:- ...)` are **renamed** when the clause is stored to prevent collision with other clauses.

```oql
;; Top-level: runs at push time
(= MyConfig {"version" "1.0"})

;; Clause: MyConfig from top-level is NOT available here
(:- (Start Exec Result)
    ;; MyConfig is undefined — it was in a different scope
    (= Result MyConfig))   ;; WRONG — MyConfig is unbound
```

## Symbol Renaming (gen-local-symbols)

When a clause is stored, all symbols in the body that are NOT in the head get renamed via `gen-local-symbols`. This is how OQL prevents variable name collisions between different clauses.

**Head symbols** survive renaming — they're the interface:
```oql
(:- (MyClause Arg1 Arg2 Result)
    ;; Arg1, Arg2, Result keep their names (they're in the head)
    ;; InternalVar gets renamed to a gensym (e.g., G__12345)
    (get Arg1 "key" InternalVar)
    (= Result InternalVar))
```

**Namespaced symbols** survive in namespace position only:
```oql
(:- (MyClause Exec Result)
    ;; Qo.Public.OqlApi.Page survives as a namespace prefix
    (Qo.Public.OqlApi.Page/page-by-id PageId Page)
    
    ;; But if you try to use a datastore symbol as a VALUE, it gets renamed
    (some-term Qo.Public.OqlApi.Page Result))  ;; WRONG — symbol gets renamed
```

## Datastores Must Be Redeclared Inside Clauses

Top-level `(datastore ...)` declarations don't survive into stored clause bodies. The datastore symbol gets renamed by gen-local-symbols, breaking the reference.

```oql
;; Top-level datastore — available at push time
(datastore MyDs "my/datastore/path")

;; WRONG — MyDs is renamed inside the clause body
(:- (MyClause Exec Result)
    (MyDs/some-term "hello" Result))   ;; MyDs is now G__12345, not a datastore

;; CORRECT — redeclare inside the clause body
(:- (MyClause Exec Result)
    (datastore MyDs "my/datastore/path")
    (MyDs/some-term "hello" Result))
```

This is why you see datastore declarations repeated inside clause bodies throughout the codebase — it's required, not redundant.

## Functors Don't Work Inside Clause Bodies

The `(functor ...)` built-in creates a functor object at execution time. While it can appear in clause bodies, the resulting functor cannot be used to define new clauses from within a stored clause. Use `run-page` or the public API instead.

```oql
;; This works at top level (push time)
(datastore HandlerDs "my/handler/path")
(functor "handle-event" 2 HandlerDs HandlerFunc)

;; Inside a clause, use run-page instead of trying to call functors directly
(:- (Start Exec Result)
    (Qo.Public.Api.Run/run-page Proto Page "start" [] Result))
```

## Capture Only Works With call

If you need to capture a value from one clause execution to use in another, you must use `call`. Values do NOT flow through functor invocations automatically.

```oql
;; WRONG — PageId from outer scope doesn't flow into handler
(HandlerDs/handle-event Event Result)

;; CORRECT — use call to explicitly pass values
(call HandlerFunc Event Result)
```

## The _ Symbol

Underscore `_` is special — it's always excluded from renaming and acts as a wildcard (matches anything, discards the value):

```oql
;; _ matches any app-id, any folder-id — we only care about PageId and Page
(Qo.Page/page-object _ _ PageId Page)
```

## Common Mistakes

1. **Using a top-level variable inside a clause** — it won't be bound at runtime
2. **Forgetting to redeclare datastores** — the namespace reference breaks silently
3. **Expecting functor results to persist across clauses** — use `call` for explicit passing
4. **Binding extra variables in index queries** — triggers full table scan instead of index lookup
5. **Using `println-stream` before binding `Result`** — corrupts the solution context; always bind your result first
