[< Home](../README.md) | [How-to](README.md)

# How to write scratch queries

Scratch queries are bare top-level term sequences with no clause mechanism. They are run with `qo run` or `qo query`.

## Always end with `(return [Result])`

Unlike implementation files (where clauses return via the head binding), scratch queries have no implicit return path. If you bind `Result` but never call `(return [Result])`, the query runner produces no result and the API returns a bare HTTP 500 with no error message, no stack trace, and nothing actionable.

```oql
;; WRONG — bare 500, no diagnostics
(= Result "hello")

;; RIGHT
(= Result "hello")
(return [Result])
```

## Why the bare 500 is hard to debug

The 500 has no details, so the natural debugging path is: "what specifically failed?" leads to checking error shapes, then datastore paths, then reading more code. The actual problem — a structurally incomplete query — is never on that path because the query *looks* valid. You have to step back to "is my query structurally complete?" which is a basic language concern, not a runtime concern.

If you get a bare 500 from a scratch query, check for `(return [...])` first.

---

## Declare every datastore the scratch file uses

`(datastore X "uri")` declarations are scoped to the file's execution context. They do **not** persist globally in the engine for other files to use.

Deployed clauses (defined with `:-`) work fine without a re-declaration in your scratch file because the engine serializes the URI string into the stored clause body in CouchDB. When the clause runs, the URI is already embedded. But a top-level scratch query that names the same datastore symbol is just a bare Clojure symbol until you declare it in *this* file. The engine receives a symbol, not a resolved Datastore protocol record, and the runtime dispatch fails.

**Symptom:** `No implementation of method: :query of protocol: #'omega-db.datastore/Datastore found for class: clojure.lang.Symbol`

The `clojure.lang.Symbol` in the error is the giveaway — the datastore name resolved as a symbol, not a datastore record.

**Fix:** Add `(datastore X "uri")` at the top of any scratch file that references the datastore, using the URI from the deployed `.oql` file where the datastore was originally declared.

```oql
;; WRONG — clojure.lang.Symbol error even though app.oql declares this datastore
(enable-full-scan)
(Qo.Schema.App.Data/app _ AppData)
(disable-full-scan)
(get AppData "name" AppName)
(get AppData "owner-email" OwnerEmail)
(return [AppName OwnerEmail])

;; RIGHT — declare the datastore at the top of this scratch file
(datastore Qo.Schema.App.Data "omega/query-omega/schema/app/data")
(enable-full-scan)
(Qo.Schema.App.Data/app _ AppData)
(disable-full-scan)
(get AppData "name" AppName)
(get AppData "owner-email" OwnerEmail)
(return [AppName OwnerEmail])
```

To find the URI for a datastore, look for its `(datastore ...)` declaration in the deployed `.oql` file that owns it (e.g. `~/Development/omega/query-omega-oql/oql/app/app.oql` declares `(datastore Qo.Schema.App.Data "omega/query-omega/schema/app/data")`).

### Why this differs from stored clauses

Inside a stored clause body, you must also re-declare any datastore the clause references (the top-level `(datastore ...)` is stripped from the stored body during serialization). The same scoping rule applies in both cases — each execution context is isolated — but the symptom path is different: deployed clauses fail silently or produce no-return; scratch files fail immediately with the `clojure.lang.Symbol` protocol error.

---

## When to use a scratch OQL file vs `qo run` in a shell loop

Before writing a scratch file, ask: can the CLI express the full operation in one `qo run` call, repeated for each item in a list?

**Use a shell loop with `qo run`** when the operation is "call this adapter method for each item in a list." A shell loop is simpler, more debuggable, requires no scratch file lifecycle management, and does not leave a file behind that future agents may mistake for a permanent artifact.

```bash
# Correct: re-activate three campaigns with a shell loop
for id in campaign-1 campaign-2 campaign-3; do
  qo run create-experiment -p Installable -m run -a "[{\"campaign-id\": \"$id\"}]"
done
```

**Use a scratch OQL file** only when the CLI cannot express the operation in one command — for example, when you need multi-step logic, joins across tables, or conditional branching that depends on query results. These are computations that require OQL's term language to express at all.

| Situation | Tool |
|---|---|
| Call an adapter method for each item in a known list | Shell loop with `qo run` |
| Inspect or join data across two tables | Scratch OQL |
| Multi-step computation (query → transform → conditional write) | Scratch OQL |
| Batch-create rows with per-row derived values | Scratch OQL |
| Run the same single-method adapter call N times | Shell loop with `qo run` |

**Source:** I8 in session 2026-05-10-smartlead-schedule-error-propagation — Task 6c was initially written as a scratch file to re-activate campaigns; the user pointed out it was just `qo run create-experiment` in a loop.
