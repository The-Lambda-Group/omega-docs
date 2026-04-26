[< Home](../README.md) | [Reference](README.md)

# defterm — semantics, when to add (or not), and migration consequences

`defterm` declares a term schema with a primary key on a datastore. It is a **relatively new construct in OQL** — it was never a hard requirement, and many existing terms in the system have **no defterm**, which is the normal default state.

This doc covers the question that `datastores.md` does not: **what does the presence or absence of a defterm actually mean at the storage layer, and what happens when you add one to a term that already has data?**

> **Read this before you "add a defterm to fix a full-scan throw."** Adding a defterm to a term that already has rows is a **destructive rekey**, not an additive index. See [Adding a defterm to a term with data](#adding-a-defterm-to-a-term-with-data) below — the existing rows do not move, and they become unreachable through the term.

## Syntax recap

The shape (covered in [datastores.md](datastores.md)):

```clojure
(datastore Qo.Data.LogStream "omega/query-omega/data/log-stream")

(defterm Qo.Data.LogStream
  {"name" "log-stream-data"
   "args" ["AppId" "LogStreamId" "LogStream"]
   "primary-key" ["LogStreamId"]})
```

Names the term, lists positional args, declares which arg(s) are the primary key.

## What a defterm controls: row-id derivation

The single most important fact about `defterm` is that it changes **how each row's `_id` is generated when the term is written**.

### Without a defterm (the default)

A term without a defterm is written through the legacy bulk-insert path. The row goes to CouchDB's `_bulk_docs` endpoint with no `_id` field, and **CouchDB assigns the row id**. From the engine's perspective, the row id is opaque — the engine reads the row back through Mango selectors or view indexes, never by id.

(Implementation note: the JVM side actually attaches a random UUID before sending; the practical effect is the same — the row id is unrelated to the term's argument values.)

### With a defterm

A term with a defterm is written through the termdef path. The row id is **derived from the primary-key value**:

```
_id = "id_" + sha256(json.write_str(pkey-args))
```

The same pkey value always produces the same `_id`. This is why defterm-backed writes are upserts (the deterministic id is the upsert key), why `fetch-revs` is needed before a bulk write (to find the existing rev), and why deleted defterm rows can be cleanly resurrected with a new write (the tombstone's rev becomes the parent of the new revision — see [omega-db/docs/explanation/fetch-revs-tombstones.md](../../omega-db/docs/explanation/fetch-revs-tombstones.md)).

> The full row-id derivation and engine flow is in [omega-db/docs/explanation/defterm-row-id-derivation.md](../../omega-db/docs/explanation/defterm-row-id-derivation.md).

## Why this matters at the language level

Three downstream consequences follow from the row-id rule:

1. **Defterm pkeys behave like SQL primary keys.** Two writes with the same pkey value collide on the same row id; the second supersedes the first. Without a defterm there is no such uniqueness — repeated writes append new rows with new auto-assigned ids.
2. **Defterm-backed terms can be looked up by exact pkey at view-index speed.** The pkey is the row id, so lookups by full pkey are O(1) hash-style fetches. Off-pkey lookups still need a `write-index` (see [datastores.md](datastores.md#write-index)).
3. **Stack traces show positional pkey matches against the defterm.** When triaging a `omega/query/full-scan` throw, the engine checks the bound-arg positions in the call against the defterm's pkey arg positions — see [reading-oql-stack-traces](https://github.com/The-Lambda-Group/query-omega-oql/blob/main/docs/reference/reading-oql-stack-traces.md) §5.

## When to add a defterm

Add a defterm **at the time you create a new term** if:

- You want pkey-based upsert semantics (writing the same pkey overwrites instead of appending).
- You want the by-pkey lookup to be a cheap deterministic fetch.
- The pkey is well-defined and stable across the term's lifetime (e.g., a UUID, a stable external id).

If you are not sure, leave the defterm off. **A term without a defterm is fully functional** — the absence is normal, not a defect. You can always add `(write-index …)` declarations later for off-pkey lookups; that path does not rekey existing data.

## When NOT to add a defterm

Do **not** add a defterm to a term that **already has data** as a way of "fixing" a query problem. See [Adding a defterm to a term with data](#adding-a-defterm-to-a-term-with-data).

Do **not** read the absence of a defterm as evidence of an orphan, a missing source artifact, or a bug. Many terms in the system have never had a defterm. Absence is the language's default.

Do **not** add a defterm "just to make it more complete." Defterms exist to enforce a uniqueness/upsert contract on the pkey. If the term has no real uniqueness contract, declaring one introduces new failure modes (e.g., `KEY_UPDATE_CONFLICT` on what looks like an independent insert) without a corresponding gain.

## Adding a defterm to a term with data

This is the section that prevents destructive migrations.

If a term has been in production with **no defterm** and accumulated rows, those rows have CouchDB-assigned (random-UUID) row ids. The engine has been reading them through Mango selectors, view indexes, or the legacy term-query path — never by id.

If you then add a defterm to the source and deploy:

1. The deploy registers the new defterm in `system/term/defterm`.
2. Subsequent writes go through the **termdef path**: row id = `id_<sha256(pkey)>`.
3. Subsequent **reads** by pkey now compute the deterministic id and try to fetch it directly.

The pre-existing rows still have their old random-UUID ids. The deterministic id the engine computes from the pkey **does not match any existing row**. From the perspective of a pkey lookup, the existing data is **unreachable** — the lookup misses every row, even though they are still physically present in the database.

> **In the project author's words: "we'd lose it."**

The data is not destroyed at the CouchDB level — the rows are still there, with their old ids and revs. But every code path that goes through the term will see them as missing. From the application layer, this is indistinguishable from data loss.

### What this is NOT

This is **not** something `(refresh)` fixes. It is **not** something a redeploy fixes. It is **not** something the engine reconciles in the background. It is a content-addressing change with no automatic migration.

### What an actual defterm migration would require

Adding a defterm to a term with data is a **separate data-migration project**. The minimum shape is:

1. Read every existing row of the term into memory (or stream it).
2. Compute the new deterministic id for each row from the pkey.
3. Write each row at the new id (the termdef-path write does this if you re-insert through the term).
4. Delete the old row at the old id.

In practice this is custom OQL or a one-shot Clojure migration script, sized to the table. It is **not** a side effect of declaring the defterm.

If you find yourself reaching for "let me just add a defterm to this term," and the term already has production data, **stop and ask the user.** The question is whether the migration is justified by the contract change you are buying. Often the answer is no, and the actual fix is elsewhere — see [Practical implication: full-scan triage](#practical-implication-full-scan-triage).

## Removing a defterm from a term with data

The mirror case. If you remove a defterm from a term that has been written with deterministic ids, subsequent writes go back through the legacy path with random ids, and the existing deterministic-id rows become orphans relative to the new write path. Pkey-based reads still resolve to the deterministic id (until the cache invalidates and the engine rediscovers there is no defterm), but the upsert contract is gone — re-writing the same pkey now appends a new row instead of updating the old one.

The same migration discipline applies: removing a defterm is destructive to the upsert contract on existing rows. Do not do it without a plan.

## Practical implication: full-scan triage

When a query throws `omega/query/full-scan` and the offending term has **no defterm**, an inexperienced agent (or a reflexive read of the triage doc) might conclude: *"the term has no pkey index, so I should add a defterm with appropriate pkey."*

**This is the wrong move on a term with data, and per the [no-defterm-as-fix rule](https://github.com/The-Lambda-Group/query-omega-oql/blob/main/docs/how-to/triage-an-oql-full-scan.md#operational-rule----read-before-doing-anything) it requires user permission anyway.**

The valid fixes for a `full-scan` throw on a no-defterm term are, in priority order:

1. **Caller-restructure** — the default. Drop over-bound args from the call; bind only what an existing index covers. See [triage-an-oql-full-scan](https://github.com/The-Lambda-Group/query-omega-oql/blob/main/docs/how-to/triage-an-oql-full-scan.md) Step 5.
2. **Add a `(functor …)` + `(write-index …)`** on the existing datastore. This adds a secondary index without rekeying any existing rows. The existing rows are read via the new view as soon as it materializes. **No data migration involved.** Requires user permission per the triage doc.
3. **Bookend with `(enable-full-scan)` / `(disable-full-scan)`** at the narrowest scope, with explicit user authorization, as an interim while the proper fix is scheduled.

**Adding a defterm is not on this list.** A defterm is a content-addressing migration, not an indexing fix. If a defterm is genuinely the right contract for the term going forward, it is a separate project that includes the data-migration step above — not a one-line declaration deploy.

## Related

- [datastores.md](datastores.md) — defterm syntax and the surrounding concepts (datastore, functor, write-index).
- [omega-db/docs/explanation/defterm-row-id-derivation.md](../../omega-db/docs/explanation/defterm-row-id-derivation.md) — Engine-level mechanics: where the deterministic id is computed, where the legacy path differs, and why the migration is destructive.
- [omega-db/docs/explanation/fetch-revs-tombstones.md](../../omega-db/docs/explanation/fetch-revs-tombstones.md) — Why defterm-backed writes need to read the current rev (including tombstones) before bulk PUT.
- [omega-db/docs/explanation/index-matching-correctness.md](../../omega-db/docs/explanation/index-matching-correctness.md) — Why binding more args than the target index expects forces a Mango fallback.
- [reading-oql-stack-traces](https://github.com/The-Lambda-Group/query-omega-oql/blob/main/docs/reference/reading-oql-stack-traces.md) — Positional pkey matching and bound-arg analysis in stack traces.
- [triage-an-oql-full-scan](https://github.com/The-Lambda-Group/query-omega-oql/blob/main/docs/how-to/triage-an-oql-full-scan.md) — Diagnostic flow for `omega/query/full-scan` throws; explicitly enumerates "do not add a defterm as a fix."
