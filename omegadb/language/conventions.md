[< Language](../language.md) | [OmegaDB](../../README.md#omegadb)

# Conventions & Gotchas

## Booleans

OQL does not support boolean `false` at runtime — it will crash the engine. Always use strings:

```oql
;; GOOD
(= Result {"has-more" "true"})

;; BAD — crashes
(= Result {"has-more" false})
```

## Return Values

Return must always be an array:

```oql
(return [Result])
```

**Bind before returning.** Map literals with symbols inside `return` will return unresolved symbolic values. Always bind to a variable first:

```oql
;; BAD — returns symbolic placeholders instead of values
(return [{"status" Status "count" Count}])

;; GOOD — bind first, then return
(= Result {"status" Status "count" Count})
(return [Result])
```

## Datastore Access

Component code should only use public API endpoints (`Qo.Public.*`), never internal datastores. If a public endpoint doesn't exist for what you need, create one in the API layer.

## Index Matching

Queries must bind EXACTLY the fields in the index — no extra bound vars, no partial matches. Check index definitions before writing queries:

```oql
;; PageIndex is ["args.2"] — bind ONLY PageId
(Qo.Page/page-object _ _ PageId Page)     ;; GOOD — matches PageIndex
(Qo.Page/page-object AppId _ PageId Page)  ;; BAD — no index on [args.0, args.2]
```

## Datastores Inside Stored Clauses

Datastore declarations at the top level don't survive into stored clause bodies (functor serialization). Redeclare inside the clause:

```oql
;; GOOD — inline datastore
(:- (MyClause! Arg Result)
    (datastore LogLoc LogLocPath)
    (log-stream LogLoc Stream)
    ...)

;; BAD — top-level datastore lost in functor
(datastore LogLoc LogLocPath)
(:- (MyClause! Arg Result)
    (log-stream LogLoc Stream)  ;; LogLoc is unresolved
    ...)
```

**Note:** This only applies to non-namespace symbols (plain aliases). Namespace symbols like `Qo.Page` resolve at runtime and don't need redeclaration inside clause bodies.

## Functors Inside Clauses

The `functor` built-in does NOT work inside stored clause bodies. If you need to call a functor from a clause, use `run-page` or `emit-event` instead.
