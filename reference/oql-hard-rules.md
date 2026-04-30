[< Home](../README.md) | [Reference](README.md)

# OQL Hard Rules

> **Confidentiality: PUBLIC.** All content in this directory is public platform documentation.

These are non-negotiable rules for writing OQL. They exist because every one of them has, at some point, produced real data loss, index corruption, or silent wrong-answer bugs. An agent or developer writing OQL without these rules in mind is a liability — not a stylistic one.

Read this before writing any line of OQL. If you catch yourself about to violate one, stop and restructure the query.

## 1. Never open a `system/` datastore directly

The `system/` namespace holds subscriptions, internal runtime state, and engine bookkeeping. Opening it and querying, much less writing, can wipe critical state that the runtime depends on.

```oql
;; FORBIDDEN
(datastore X "system/...")
```

There is no user-facing reason to open `system/`. If you think you need something from it, you need a public API clause instead — ask.

## 2. Never open internal data-layer datastores directly

The internal data layer lives under `omega/query-omega/data/...`. It is the storage layer behind the public API, not a public surface. Opening it directly — or invoking `Qo.Data.*/` terms — bypasses access control, schema guarantees, and every invariant the public API enforces.

```oql
;; FORBIDDEN
(datastore X "omega/query-omega/data/...")

;; FORBIDDEN
(Qo.Data.Prop/some-term ...)
```

Use library-level clauses instead. These are the sanctioned public API:

- `Qo.Db.Prop/` — property system (schema, keys, indexes, table writes)
- `Qo.Page/` — page tree navigation and mutation
- `Qo.Public.OqlApi.*/` — public API surface (see [api.md](api.md))

If the public API lacks the operation you need, that is a gap to file — not a license to reach past it.

**Read-only audit exception — `omega-cli db-name` + CouchDB HTTP.** For pre-migration row counts and audits, the sanctioned alternative is NOT a direct OQL `Qo.Data.*/` call — it is `omega-cli db-name <path> | curl GET /<db>/` to read `doc_count` from CouchDB metadata. This stays fully outside OQL and therefore outside Rule 2's scope. See [how-to/count-rows-in-a-defterm.md](../how-to/count-rows-in-a-defterm.md).

## 3. Never run mutations in scratch files

Scratch files (invoked via `qo run` or `qo query` on a bare top-level term sequence) are **read-only**. No `delete!`, no `refresh`, no `write!`, no `!`-suffixed term that performs a write.

Mutations belong in deployed implementations, invoked via `qo run` / `qo push`:

1. Write the mutation as a clause in an implementation file.
2. `qo push` the file.
3. `qo run` the deployed implementation.

This discipline exists because scratch files have no review trail, no implementation identity, and no protocol scoping. A mutation that lands from a scratch file is untraceable; the same mutation in a deployed implementation has a file, a push record, and a replayable run.

See [write-scratch-queries.md](../how-to/write-scratch-queries.md) for the correct use of scratch files (read-only inspection, returning bound values for dry-run verification).

## 4. Never use `refresh` on any datastore

`refresh` wipes the datastore and breaks the CouchDB indexes that back it. The indexes do not silently rebuild — subsequent queries against the datastore return wrong answers or fail outright, and recovery requires rebuilding indexes by hand.

There is no legitimate workflow that calls `refresh`. If you want to remove specific rows, use targeted `delete!` from a deployed implementation (rule 3). If you want to reset a development fixture, drop and recreate the property via the public API, not `refresh`.

## 5. Never use boolean `true` / `false` in OQL — use strings

OQL's solution engine handles native booleans incorrectly in ways that produce **silent wrong answers** — no error, no warning, just a query that returns the wrong rows. Always use the strings `"true"` and `"false"` instead.

```oql
;; FORBIDDEN — silent wrong answers
(= Flag true)
(get Map "enabled" true)

;; CORRECT
(= Flag "true")
(get Map "enabled" "false" Flag)
```

This applies everywhere: term arguments, map values, comparisons, defaults. If a downstream system requires a real boolean (e.g. a JSON payload to an HTTP API), construct it at the boundary via `json-stringify` on a map whose values are already the correct JSON primitive — never introduce a literal `true`/`false` into the OQL solution space.

## Why these are hard rules

These are not stylistic preferences. Each rule maps to a failure mode observed in production:

- **Rule 1** — direct `system/` access has wiped runtime subscription state.
- **Rule 2** — direct `Qo.Data.*/` writes have bypassed schema invariants and corrupted rows the public API later could not read.
- **Rule 3** — scratch-file mutations have deleted data with no trail back to the query that did it.
- **Rule 4** — `refresh` has broken indexes and left datastores returning partial results until manual rebuild.
- **Rule 5** — boolean `true`/`false` has produced silent wrong-answer bugs that took hours to diagnose because the query looked correct.

Treat a temptation to violate any of these as a sign that you are solving the wrong problem.
