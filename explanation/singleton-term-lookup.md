[< Home](../README.md) | [Explanation](README.md)

# Singleton-row term lookup — why no index applies

Some terms hold **exactly one row** by design — a global config, a singleton service descriptor, a per-cluster invariant. The natural call shape for reading that row is arity-1 with the **single arg as the output**:

```clojure
(Qo.Schema.Schema/config-data Config)
```

There is no input arg to constrain on, no parent id to look up by, no key to bind. The caller wants "the row" and there is exactly one.

This shape sits at the seam of the engine's index-matching model and currently has **no clean answer** in OQL — it is the reason `Qo.Schema.Schema/create-config!` retains an `(enable-full-scan)` / `(disable-full-scan)` bookend after the rest of the bookend migration was retired (see `query-omega-oql/oql/app/schema/schema.oql:90-96`).

This doc explains why.

## What the engine does at index-match time

The OQL engine picks an index by inspecting **which arg positions are bound** at the call site, then matching that bound-arg set against the term's primary-key positions and any declared `write-index` field sets. The rule is "bound args ⊆ index field set" (see [index-matching-correctness](../../omega-db/docs/explanation/index-matching-correctness.md)).

For an arity-1 term where the **only** arg is the output:

- Bound-arg set: **empty** (`{}`).
- pkey field set on the defterm (if any): a non-empty list of arg names — pkey requires bound positions to compute the deterministic id.
- `write-index` field sets: each declares one or more arg positions; an empty bound-arg set matches **none** of them.

The empty bound-arg set is a subset of every declared field set, but the engine does not match on subset-of-empty. It matches on **exact field-set coverage** and falls through to Mango when no declared index has its keys bound. With nothing bound, no view-key prefix can be constructed, so no view query is possible — Mango is the only fallback.

That fallback fires the `omega/query/full-scan` throw, and the only way to read the row through the term today is to bookend the call with `(enable-full-scan)` / `(disable-full-scan)`.

## Why a defterm doesn't help

The reflex from [triage-an-oql-full-scan](https://github.com/The-Lambda-Group/query-omega-oql/blob/main/docs/how-to/triage-an-oql-full-scan.md) — "if no index serves this query, ask whether one is missing" — does not produce an answer in the singleton case.

A defterm declares a primary key that becomes the row's deterministic id (see [defterm](../reference/defterm.md)). The id is `id_<sha256(json.write_str(pkey-args))>`. For that to be useful as a lookup key:

1. The pkey arg list must be **non-empty** — `sha256(json.write_str([]))` is a single fixed digest, but the engine has no convention for "use the empty pkey to fetch the singleton."
2. The caller has to **bind** the pkey args at lookup time so the engine can compute the deterministic id. With an output-only call shape, nothing is bound.

A defterm with `"primary-key" []` is not a documented or supported declaration — it would require engine support to recognize the empty pkey as a known sentinel and route lookups through a single deterministic id. That work has not been done.

A `write-index` does not help either: a `write-index` declares which arg positions to materialize a view over. With no bound args, every view's start-key is empty and the runtime cannot pick one over another — there is no way to express "the view that yields the one row" through the existing field-set match.

## What the call shape means semantically

The singleton-row term is the OQL-language analogue of a SQL query like:

```sql
SELECT config FROM schema_config; -- exactly one row
```

In SQL the shape is unambiguous because tables are not the same construct as terms — a table read with no `WHERE` clause is a sequential scan but the planner knows the cardinality. OQL's term/index model assumes every read is keyed on something the caller binds. The singleton case is the corner where that assumption breaks.

There is no caller-restructure fix (the canonical [triage step 5](https://github.com/The-Lambda-Group/query-omega-oql/blob/main/docs/how-to/triage-an-oql-full-scan.md#step-5----fix-the-caller)) because the caller is not over-binding — it is **zero-binding**. The pattern in [lookup-with-consistency-check](https://github.com/The-Lambda-Group/query-omega-oql/blob/main/docs/how-to/lookup-with-consistency-check.md) — "bind only pkey, AND in the consistency check" — assumes the caller has *something* to bind. In the singleton case there is nothing.

## Worked example: `Qo.Schema.Schema/config-data`

`Qo.Schema.Schema` is the global schema-config datastore. It holds **one row** — the merged result of every per-component `create-config` call — and that row is the platform's runtime view of its own schema.

The clauses around it (`query-omega-oql/oql/app/schema/schema.oql:30-96`) compose the row from per-namespace fragments and write it once:

- `init-config-data!` builds the merged `Config` map by calling each `Qo.Schema.<ns>/create-config` and folding the results.
- `init-config!` writes it (or updates the existing row).
- `create-config!` reads it back. **This is the singleton lookup.**

The read shape is:

```clojure
(:- (Qo.Schema.Schema/create-config! Config)
    (enable-full-scan)
    (Qo.Schema.Schema/config-data Config)
    (disable-full-scan))
```

The bookend stays because:

1. There is no input arg. The single arg `Config` is the output.
2. There is no other access pattern that could be added as an index — the row has no externally-meaningful field to key on; it *is* the schema, looked up by the fact that there is exactly one.
3. Adding a defterm would require both engine support for an empty pkey and a data migration of any existing `Qo.Schema.Schema` row to the deterministic id (see [defterm — adding to a term with data](../reference/defterm.md#adding-a-defterm-to-a-term-with-data)).

The bookend is a placeholder for a missing engine pattern, not an over-binding fix that was deferred.

## Adjacent shapes that are NOT this case

Do not reach for "singleton lookup" framing when the term actually has bound input args available. In particular:

- **By-id lookup of a known row.** If the caller already has the row's id, pkey, or any indexed field, bind that field and the term reads through the existing index. Not this case.
- **Per-parent singleton.** A term that holds one row *per parent* (e.g., one config row per AppId) is a normal pkey-on-parent-id lookup — bind the parent id, hit the pkey index. Not this case.
- **Cardinality-1 by accident.** A term that *happens* to have one row in a particular environment but is not architecturally singular has a normal index; the call should bind whatever the index covers. Not this case.

The singleton case requires **architectural cardinality 1** — the term schema enforces one row by construction, and the read has no input arg available because there is no input dimension to bind.

## What a future fix would look like

Resolving the singleton case requires engine support. The shape is sketched here to make the gap concrete; none of this is implemented.

1. **Empty-pkey defterm support.** Allow `"primary-key" []` in a defterm declaration. The deterministic id becomes a single fixed digest (`id_<sha256("[]")>` or a reserved sentinel). Writes upsert to that id; reads with a fully-free arg list compute the same id and fetch directly. Would also need the bulk-insert path to recognize the empty pkey case so legacy random-id rows do not silently shadow the singleton row.

2. **A by-singleton term call shape.** A separate engine convention — for example, a built-in `(singleton-of <Term> Result)` — that bypasses the index-match step entirely and reads the cardinality-1 row by a known reserved id. This avoids stretching `defterm` semantics but requires a new language surface.

3. **Migration of existing data.** Any term that already has a singleton-shaped row written through the legacy path has a CouchDB-assigned random id. Switching to either of the above would orphan the existing row exactly the same way that adding a defterm to a populated term does (see [defterm](../reference/defterm.md#adding-a-defterm-to-a-term-with-data)). The singleton case would need the same one-shot rekey.

Until one of these lands, the bookend is the working answer for `Qo.Schema.Schema/config-data` and any future term with the same shape. Mark such bookends in source with a comment pointing back to this doc so they can be retired together when the engine pattern arrives.

## Related

- [defterm](../reference/defterm.md) — Why pkey field-set governs the deterministic id, why empty pkey is not a documented case, and why adding a defterm to a populated term is destructive.
- [omega-db/docs/explanation/index-matching-correctness.md](../../omega-db/docs/explanation/index-matching-correctness.md) — The "bound args ⊆ index field set" rule that the singleton case sits outside of.
- [triage-an-oql-full-scan](https://github.com/The-Lambda-Group/query-omega-oql/blob/main/docs/how-to/triage-an-oql-full-scan.md) — Diagnostic flow for `omega/query/full-scan` throws. The singleton case is the one path through that flow where neither caller-restructure nor a new index applies, and the bookend is justified — but only with explicit user authorization and a marker comment.
- [lookup-with-consistency-check](https://github.com/The-Lambda-Group/query-omega-oql/blob/main/docs/how-to/lookup-with-consistency-check.md) — The default pkey-only lookup pattern for non-singleton terms; contrast with the zero-binding singleton case.
- [reading-oql-stack-traces](https://github.com/The-Lambda-Group/query-omega-oql/blob/main/docs/reference/reading-oql-stack-traces.md) — Bound vs free variables and the Mango selector dict; in a singleton-case throw, `selector.args` is empty (no numeric-keyed positions, only `$exists: true` scaffolding).
