[< Home](../README.md) | [Reference](README.md)

# Prefix-Scan Index Selection and Operator Push-Down

This document covers how `query-term-from-table` (inside `omega-db.datastore.couchdb.data_types.query`) selects an execution strategy when a term query is evaluated, and how the OQL operators `with-sort`, `with-skip`, and `with-limit` behave under each strategy.

> **Why this matters:** `with-skip` and `with-limit` are silently not pushed to the database in two of the four strategies. A clause that uses them without knowing which strategy is active will produce quietly incorrect results.

---

## The four strategies

### Strategy 1 — Defterm PK lookup

**When it fires:** The bound columns exactly match the term's declared primary key (all PK fields are bound, in order).

**Mechanism:** The engine computes the deterministic row ids from the PK values and fetches them in a single `_all_docs` POST. No view index is used.

**Operator push-down:**

| Operator | Pushed? |
|---|---|
| `with-limit` | **No** |
| `with-skip` | **No** |
| `with-sort` | **No** |

All three operators are applied (or silently ignored) after the fetch. Because PK lookup usually returns at most one row per key, this is rarely consequential — but if the caller passes a multi-key batch, skip/limit will not filter at the CouchDB layer.

---

### Strategy 2a — Exact write-index bind

**When it fires:** A `write-index` covers the bound positions, AND the number of bound positions equals the total number of fields in the index (i.e., every index field is bound).

**Mechanism:** A single POST with `{:keys ks}` against the view. Every row returned is a direct hit.

**Operator push-down:**

| Operator | Pushed? |
|---|---|
| `with-limit` | **Yes** — threaded as `limit` in the CouchDB view request |
| `with-skip` | **Yes** — threaded as `skip` |
| `with-sort "desc"` | **Yes** — `descending=true` in the view request |

This is the highest-performance path and the only one where all three operators are fully pushed down.

---

### Strategy 2b — Prefix bind, one bound row

**When it fires:** A `write-index` covers the bound positions as a prefix (the first N fields of the index are bound; the remaining fields are unbound), AND there is exactly **one** bound row (i.e., one combination of bound-arg values).

**Mechanism:** A range query using `startkey` / `endkey`. The endkey sentinel is:

```
(conj (vec bound-key) {})
```

CouchDB sorts `{}` (an empty map) after all scalar values, so appending it to the key vector closes the range just beyond the last matching row. For descending sort, `startkey` and `endkey` are swapped.

**Operator push-down:**

| Operator | Pushed? |
|---|---|
| `with-limit` | **Yes** |
| `with-skip` | **Yes** |
| `with-sort` | **Yes** |

All three operators are pushed to the view, same as strategy 2a.

---

### Strategy 2c — Prefix bind, N bound rows

**When it fires:** A `write-index` covers the bound positions as a prefix, AND there are **two or more** distinct bound rows (multiple combinations of bound-arg values).

**Mechanism:** The engine issues one `lazy-range-fetch` per bound row (each is a separate CouchDB POST for a range, using `startkey_docid` for chunked pagination within a single range at a default chunk size of 1 000 rows). The per-range streams are then merged into a single globally-sorted stream using a **k-way merge heap** (`omega-db.datastore.couchdb.heap-merge/k-way-merge`).

The heap is a priority-map keyed by the sort-column value (min-heap), holding one head row per non-empty range. Memory cost is O(N) where N is the number of ranges, not the total row count.

**Operator push-down:**

| Operator | Pushed? |
|---|---|
| `with-limit` | **No** — applied AFTER merge |
| `with-skip` | **No** — applied AFTER merge |
| `with-sort` | **Partially** — drives the merge key-fn, not pushed to individual view requests |

This is the strategy most likely to produce unexpected performance or correctness issues. If a query binds multiple values across two dimensions and a secondary index covers only the first dimension as a prefix, OQL will silently issue one range fetch per outer-dimension value and merge them in memory. `with-limit 10` will still return only 10 rows, but the database may scan far more before the merge produces 10 merged results.

---

### Strategy 3 — Mango fallback

**When it fires:** No `write-index` covers the bound positions as a prefix, AND `(enable-full-scan)` is active in the clause scope.

**Mechanism:** A CouchDB Mango query against the raw document store, with no view involved.

**Operator push-down:**

| Operator | Pushed? |
|---|---|
| `with-limit` | **No** |
| `with-skip` | **No** |
| `with-sort` | **No** |

None of the operators are pushed. The Mango path returns up to the page-query default ceiling (effectively unbounded). This is covered in more detail in [query-execution.md](query-execution.md).

---

### Strategy 4 — No-matching-index error

**When it fires:** No `write-index` covers the bound positions as a prefix AND `enable-full-scan` is **not** active.

**Mechanism:** The engine throws immediately:

```clojure
{:type "omega/query/no-matching-index"
 :bound-positions [...]
 :candidate-indexes [...]}
```

`:bound-positions` lists the `args.N` strings the caller bound. `:candidate-indexes` lists the index field lists the engine considered but rejected. This error is the safest outcome when no index covers the query — it surfaces the problem explicitly rather than silently scanning.

---

## Index selection algorithm

**Prefix test:** An index qualifies as a candidate if its `:fields` vector **starts with** the bound `args.N` strings in order. For example, if the bound positions are `["args.1"]` and an index has fields `["args.1" "args.2"]`, the index qualifies (args.1 is a prefix of [args.1, args.2]).

**Candidate ranking:** Among all qualifying candidates, the engine selects the one with the **shortest field list** — the closest match to the bound positions. If two indexes have the same length, the behavior is implementation-defined (do not rely on a specific winner).

**Defterm PK priority:** The PK-exact-match path (Strategy 1) is checked before the write-index paths. If the bound columns exactly satisfy the PK, the PK path is taken regardless of any write-indexes.

---

## Operator push-down summary table

| Strategy | `with-limit` | `with-skip` | `with-sort` |
|---|---|---|---|
| 1 — Defterm PK lookup | Not pushed | Not pushed | Not pushed |
| 2a — Exact write-index | Pushed | Pushed | Pushed |
| 2b — Prefix bind, 1 row | Pushed | Pushed | Pushed |
| 2c — Prefix bind, N rows | **Not pushed** | **Not pushed** | Drives merge |
| 3 — Mango fallback | Not pushed | Not pushed | Not pushed |
| 4 — No-index error | — | — | — |

---

## Heap-merge primitives (Strategy 2c)

**`lazy-range-fetch`** (in `omega-db.datastore.couchdb.heap-merge`):

- Wraps the CouchDB view POST for one startkey/endkey range.
- Uses `startkey_docid` as a pagination cursor within the range, fetching in chunks of 1 000 rows by default.
- Returns a lazy sequence; rows are fetched from CouchDB on demand as the merge consumes them.

**`k-way-merge`** (in `omega-db.datastore.couchdb.heap-merge`):

- Priority-map (min-heap) keyed by the sort-column value.
- Holds exactly one head row per non-empty range.
- Memory cost: O(K) where K is the number of ranges (not the total row count).
- Produces a globally sorted stream by repeatedly popping the minimum-key row and advancing that range's cursor.

---

## Endkey sentinel

For prefix-scan strategies (2b and 2c), the endkey is:

```clojure
(conj (vec bound-key) {})
```

An empty map `{}` sorts **after all scalar values** in CouchDB's collation order, so appending it closes the range immediately after the last row whose key starts with `bound-key`. This avoids the need to compute a next-sibling key.

For **descending** sort, `startkey` and `endkey` are swapped (CouchDB descending ranges are `[startkey, endkey)` where `startkey > endkey`).

---

## Related

- [query-execution.md](query-execution.md) — `with-skip`/`with-limit` silent-ignore on Mango fallback (Strategy 3 detail).
- [datastores.md](datastores.md) — `write-index` declaration syntax and how secondary indexes are registered.
- [defterm.md](defterm.md) — How defterm primary keys are derived and why they enable Strategy 1.
- [reducers.md](reducers.md) — `with-order-by`, `with-skip`, `with-limit` in the OQL reducer layer.
