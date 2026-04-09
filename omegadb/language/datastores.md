[< Language](../language.md) | [OmegaDB](../../README.md#omegadb)

# Datastores, Functors & defterm

## Datastores

Data in OQL lives in **datastores**. A datastore is a named location that holds terms (facts and rules). You declare a datastore and then use it as a namespace prefix:

```oql
(datastore Qo.Page "omega/query-omega/page")

;; Query the page datastore
(Qo.Page/page-object _ _ PageId Page)
```

The string `"omega/query-omega/page"` is the datastore's path in the database. The symbol `Qo.Page` is how you refer to it in the query.

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
