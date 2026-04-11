[< Home](../README.md) | [Reference](README.md)

# Clauses

A clause defines a rule — when its head is true based on its body. Clauses are defined with `(:- ...)`:

```oql
(:- (Head Arg1 Arg2 Result)
    (some-term Arg1 Intermediate)
    (other-term Intermediate Result))
```

Read this as: **Head is true if the body terms are all true.**

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

;; You can call it in the same query
(ParseSales "100" Result)   ;; Result = 100
```

In-memory clauses are used for:
- **Helper logic** inside an implementation (parsing, scoring, transforming)
- **Implementation clauses** that get registered with `set-implementation-clauses`
- **Captured clauses** passed into stored clauses via the `capture` mechanism

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
    (ParseValue RawVal Parsed)
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
