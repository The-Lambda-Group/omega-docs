[< Home](../README.md) | [Gotchas](README.md)

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

## What Survives Into a Clause Body

When a clause is stored or invoked, only two things make it into the clause body's solution space:

1. **Head symbols** — the arguments declared in the clause head. These are the interface.
2. **Namespaced datastore references** — symbols used with namespace syntax (e.g., `Qo.Page/page-object`). The datastore reference is preserved because it's in namespace position.

**Everything else is renamed.** All other symbols in the body get replaced with gensyms (e.g., `G__12345`) via `gen-local-symbols`. This prevents variable name collisions between different clauses.

```oql
(:- (MyClause Arg1 Arg2 Result)
    ;; Arg1, Arg2, Result — SURVIVE (head symbols)
    ;; Qo.Page — SURVIVES (namespace reference in Qo.Page/page-object)
    ;; InternalVar — RENAMED to G__12345
    ;; Page — RENAMED to G__12346
    (Qo.Page/page-object _ _ Arg1 Page)
    (get Page "name" InternalVar)
    (= Result InternalVar))
```

**Namespaced symbols only survive in namespace position**, not as values:
```oql
(:- (MyClause Exec Result)
    ;; Qo.Page survives here — it's in namespace position
    (Qo.Page/page-object _ _ PageId Page)
    
    ;; Qo.Page gets RENAMED here — it's used as a value argument
    (some-term Qo.Page Result))  ;; WRONG — Qo.Page is renamed to a gensym
```

## Datastores in Clause Bodies

Since namespaced references survive in namespace position, datastores declared at the top level *do* work inside clause bodies — **but only when used with namespace syntax**:

```oql
(datastore Qo.Page "omega/query-omega/page")

;; This WORKS — Qo.Page is used in namespace position
(:- (MyClause PageId Result)
    (Qo.Page/page-object _ _ PageId Result))
```

However, if you need to use a datastore symbol as a value (not in namespace position), or if you're building a datastore path dynamically, you must redeclare it inside the clause body:

```oql
;; CORRECT — dynamic datastore path built inside clause
(:- (MyClause AppId LogStreamId Result)
    (string-split LogPath "/" ["omega/query-omega/apps" AppId "log-stream/streams" LogStreamId])
    (datastore LogLoc LogPath)
    (log-stream LogLoc Stream)
    ;; ...
    )
```

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
