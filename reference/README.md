[< Home](../README.md)

# Reference

> **Confidentiality: PUBLIC.** All content in this directory is public platform documentation.

Factual lookup for Omega APIs, schemas, and IDs. Read these when you need exact signatures, protocol definitions, or command inventories. These are "what is ..." docs -- if you need "how do I ..." task recipes, see [how-to/](../how-to/README.md); if you need "why does ..." understanding, see [explanation/](../explanation/README.md).

## Information ownership

This bucket owns the public OQL language reference and public API surface. Internal datastore schemas and engine internals are documented in `omega-knowledge-base`. CLI command reference lives in `query-omega-cli/docs/reference/`.

## Navigation map

### Language reference

- [OQL hard rules](oql-hard-rules.md) -- Non-negotiable rules an agent must never violate when writing OQL. Covers forbidden datastore opens (`system/`, `omega/query-omega/data/...`, `Qo.Data.*/`), forbidden operations in scratch files (any `!`-suffixed write, `delete!`, `refresh`), the absolute prohibition on `refresh` (breaks CouchDB indexes), and the string-vs-boolean requirement (`"true"`/`"false"` only -- native `true`/`false` produces silent wrong answers). Read before writing any line of OQL. Keywords: hard rules, forbidden, system datastore, internal datastore, Qo.Data, scratch mutations, refresh, boolean, true, false, silent wrong answer.

- [Clauses](clauses.md) -- OQL clause syntax and semantics. Covers stored clauses (the `!` read/write convention with datastore namespaces), in-memory clauses (local function-like definitions), inner clauses (experimental, with known limitations on stored writes), and closures/capture (threading values across clause boundaries with the `{"capture" [...]}` syntax). Keywords: clause, `:-`, stored clause, `!` suffix, in-memory clause, inner clause, capture, closure, functor.

- [Built-in terms](built-ins.md) -- The complete built-in term inventory. Covers core terms (`=`, `defined`, `return`, `uuid`, `time-now`), arithmetic (`+`, `-`, `*`, `/`, `>`, `<`, `>=`), map operations (`get`, `get-in`, `set`, `merge`, `select-keys`, `zip`), array operations (`get-list`, `length`, `append`, `concat`, `partition`), string operations (`string-split`, `string-concat`, `string-contains`, `string-replace`, `re-find`, `json-stringify` including the bidirectional parse/stringify contract and the parse-probe pattern where wildcard `_` fails but a named symbol succeeds), data transformation patterns (preparing data for `write-table`), the symbols-in-literal-arguments gotcha (why you must bind maps with `=` before passing them to terms), and the HTTP term (`http-request` with request/response map keys, plus deprecated `http-get-request`/`http-post-request`). Keywords: built-in, term, unify, get, set, merge, zip, get-list, json-stringify, json-parse, parse probe, wildcard, string-concat, http-request, http-get-request, symbols in literals.

- [Control flow](control-flow.md) -- The most critical reference doc. Covers universal quantifiers (`when`, `when-not`, `if` -- these are "if any" operators, not per-row), per-solution branching (`with-table-if` -- condition evaluation, schema/binding-list as JOIN key, branch scoping), preserving per-row identity (the `SELECT DISTINCT` collapse problem and why schemas need unique-per-row columns), dropping rows from solutions (`(= true false)` idiom), batch-wide vs per-row conditions (`defined` is lexical, not per-row), nesting costs and the flat-sequential alternative, the check-then-rebind idiom, the Ctx-threading pattern, binding-list association rules, and the absence of a `not` operator. Keywords: when, when-not, if, with-table-if, per-row, universal, schema, binding list, row collapse, identity column, `(= true false)`, drop rows, defined, not operator, Ctx, scope, JOIN key.

- [Datastores](datastores.md) -- Datastore declarations, functors, defterm, and write-index. Covers how `(datastore Name "path")` binds a symbol to a CouchDB location, how `(functor "name" arity Ds Func)` creates callable references, how `(defterm Ds {...})` declares term schemas with primary keys to create indexes, and how `(write-index Func "path" ["args.N"])` creates secondary indexes. Keywords: datastore, functor, defterm, write-index, primary key, secondary index, CouchDB, term schema, arity.

- [Reducers](reducers.md) -- Solution table aggregation operations. Covers `fold` (solution-wide reduce with init/sym/func/agg), `with-unique-by` (changing what "unique" means for fold -- essential when fold symbols have few distinct values), `with-order-by`, `with-group-by`, `with-count`, `with-limit` (not yet implemented -- will crash). Includes critical pitfall documentation: fold is solution-wide not per-row, fold folds over unique values not rows, and why `batch=1` masks fold bugs. Keywords: fold, reduce, aggregate, with-unique-by, with-order-by, with-group-by, with-count, with-limit, append, per-row vs solution-wide.

- [Query execution](query-execution.md) -- The `with-skip`/`with-limit` constraint for CouchDB view pagination. Covers the critical limitation that skip/limit only work when the runtime resolves through a view index -- when it falls back to Mango scan, skip/limit are silently ignored and the query returns up to 1,000,000 rows. Keywords: with-skip, with-limit, pagination, view index, Mango scan, silent bug, page-query, CouchDB.

### API & protocols

- [Public API](api.md) -- Complete clause signature tables for all public OQL API datastores. Covers `Qo.Public.OqlApi.Page` (page tree navigation/mutation), `Qo.Public.OqlApi.Page.Bl.LogStream` (log stream blocks), `Qo.Public.OqlApi.Page.Bl.PushCon` (push connector blocks), `Qo.Public.OqlApi.Comp` (component definitions), `Qo.Public.OqlApi.Proto` (protocol definitions), `Qo.Public.OqlApi.Impl` (implementation management), `Qo.Public.OqlApi.Db.Prop` (database property system -- schema, primary keys, indexes, table writes, truncation), `Qo.Public.OqlApi.PageSub` (page event subscriptions), `Qo.Public.Api.Run` (run implementations by page reference/ID), and `Qo.Public.Api.LogStream` (read log streams). Keywords: public API, Qo.Public, clause signature, page, component, protocol, implementation, database, property, subscription, run, log stream.

- [Protocols](protocols.md) -- Protocol IDs, JSON specs, and usage examples for all four standard protocols: Installable (`install` -- one-time setup), Service (`start`/`stop`/`status` -- lifecycle management), Event Handler (`on-event` -- event dispatch), and Query (`run` -- stateless query execution). Includes `qo run` invocation examples for each. Keywords: protocol, protocol ID, Installable, Service, Event Handler, Query, install, start, stop, status, on-event, run, qo run.
