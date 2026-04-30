[< Home](../README.md) | [How-to](README.md)

# Count all rows in a defterm

> **Confidentiality: PUBLIC.** All content in this file is public platform documentation.

This is the recommended approach for **pre-migration audits**, **sanity checks**, and any other situation where you need a total row count for a defterm table. It does not require a running OQL query.

## The recommended path: CouchDB `doc_count`

Every defterm-backed datastore is a CouchDB database. CouchDB maintains `doc_count` as O(1) view metadata — reading it costs one HTTP call regardless of table size.

### Step 1 — Find the CouchDB database name

Use `omega-cli db-name` with the datastore path (the string in the `datastore` declaration, not the OQL namespace symbol):

```bash
omega-cli db-name "omega/query-omega/data/page/block"
# → db_724c857a57fa48118808fe48543308c392144fdb1afe1a4aae3371a792e7cb5c
```

The datastore path for a defterm lives in the source `.oql` file. For the Page/Block defterm:

```oql
(datastore Qo.Data.Page.Block "omega/query-omega/data/page/block")
(defterm Qo.Data.Page.Block
  {"name" "page-block"
   "args" ["AppId" "PageId" "BlockId" "Block"]
   "primary-key" ["BlockId"]})
```

So the path is `"omega/query-omega/data/page/block"`.

### Step 2 — Read `doc_count` from CouchDB

```bash
DBNAME=$(omega-cli db-name "omega/query-omega/data/page/block")
curl -s -u admin:$COUCHDB_PASSWORD "http://localhost:5984/$DBNAME/" | jq .doc_count
# → 588
```

`doc_count` is the live document count. Deleted rows (tombstones) are reported separately as `doc_del_count`.

**Empirically verified 2026-04-30:** `doc_count: 588` for the Block table on a local omega-db instance.

### Environment

- **Local:** `http://localhost:5984/` — `COUCHDB_PASSWORD` is in `~/Development/omega/omega-db/.env`
- **Production:** use the production CouchDB host and credentials

---

## Why not `with-count`?

`with-count` is the OQL primitive for counting rows. Its documented usage (`reducers.md § with-count`) does not require a bound argument — you can pass wildcards:

```oql
;; DOCUMENTED FORM — expected to count all rows
(with-count Cnt (Qo.Page.Bl/page-block _ _ _ _))
(return [Cnt])
```

**In practice, this currently NPEs** (empirically confirmed 2026-04-30, local omega-db):

```
omega/jvm/NullPointerException
failed at: ((with-count Cnt (Qo.Page.Bl/page-block _ _ _ _)))
```

This is a fourth known `with-count` NPE pattern — all-wildcard defterm call — beyond the three documented in [gotchas/with-count-captured-helper-npe.md](../gotchas/with-count-captured-helper-npe.md). Whether it shares a root cause with the other three is **Unknown**. Engine maintainers should investigate.

**Until the NPE is resolved: use the CouchDB-direct approach above** for any all-defterm row count.

---

## When `with-count` does work

`with-count` works reliably when:

- The inner term is a **single clause call** (not compound `and`)
- At least the **pkey argument is bound** to a specific value
- The call does **not** invoke a captured helper or a helper containing `prop-vals-by-folder`

Example — count all rows for a specific folder ID (from `reducers.md`):

```oql
(with-count Total (Qo.Db.Prop/prop-vals FolderId _ _))
```

Here `FolderId` is bound, matching the folder-index path. This form works. The all-wildcard form does not.

### Decision table

| Goal | Method |
|------|--------|
| Count **all rows** in a defterm table (audit, migration preflight) | `omega-cli db-name` + `curl GET /<db>/` → `doc_count` |
| Count rows **matching a bound pkey** (application code) | `with-count` with pkey bound — standard form |
| Count rows matching a **compound predicate** (captured helper) | `with-table-if` + `fold`-per-row-unique + `length` — see [with-count-captured-helper-npe.md](../gotchas/with-count-captured-helper-npe.md) |

---

## Cross-references

- [reference/reducers.md § with-count](../reference/reducers.md#with-count) — `with-count` semantics, NPE patterns, and the fold-based workaround
- [gotchas/with-count-captured-helper-npe.md](../gotchas/with-count-captured-helper-npe.md) — three confirmed `with-count` NPE patterns and the canonical workaround
- [reference/datastores.md](../reference/datastores.md) — `datastore`, `defterm`, and `write-index` declarations (where to find the datastore path string)
- [reference/query-execution.md](../reference/query-execution.md) — Mango-fallback rule (applies to `with-skip`/`with-limit`; does **not** apply to `with-count` — `with-count` reads CouchDB view metadata, not a view scan)
