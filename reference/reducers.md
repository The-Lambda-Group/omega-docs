[< Home](../README.md) | [Reference](README.md)

# Solution Table Operations (Reducers)

These operations work on the solution table — they aggregate, filter, or reorder rows across the solution set.

```oql
(fold Init Sym Func Agg)                  ;; reduce across solutions
(with-count Count Term)                   ;; count solutions matching a term
(with-order-by [Sym] Body)               ;; order solutions by symbol value
(with-group-by [Sym] Body)               ;; group solutions by symbol value
(with-unique-by [Sym] Body)              ;; deduplicate solutions by symbol value
(with-limit N Body)                       ;; limit to first N solutions
(with-skip N Body)                        ;; skip first N solutions
(with-sort Dir Body)                      ;; sort direction: "asc" or "desc"
```

## fold

Reduces a symbol across all solutions in the table. The function is applied pairwise (like `reduce` in functional programming):

```oql
;; Collect all names into an array
(get Items _ Item)
(get Item "name" Name)
(fold [] Name append Names)              ;; Names = ["Alice", "Bob", "Carol"]

;; Sum a numeric value
(get Items _ Item)
(get Item "score" Score)
(fold 0 Score + Total)                   ;; Total = sum of all scores
```

`fold` takes four arguments:
1. **Init** — initial accumulator value (`[]` for arrays, `0` for sums)
2. **Sym** — the symbol to reduce over (one value per solution row)
3. **Func** — the reducer function (`append`, `+`, `*`, or a custom clause)
4. **Agg** — the output symbol that holds the final result

## with-unique-by

OQL solution tables are unique by default — each row represents a unique combination of all bound symbols. `fold` reduces over these unique solutions, which means it deduplicates by the fold symbol automatically. For example, `(fold 0 Score + Total)` only sums unique values of `Score`.

`with-unique-by` changes what "unique" means for folds inside its body. Instead of deduplicating by the fold symbol, it deduplicates by the specified symbol(s). This is essential when you have a symbol with few distinct values (like a flag that's `0` or `1`) but need to sum it per-row rather than per-unique-value.

```oql
;; Problem: ApiCall is 0 or 1 per row. Without with-unique-by,
;; fold sees at most 2 unique values of ApiCall (0 and 1),
;; so ApiCount can never exceed 1.
(fold 0 ApiCall + ApiCount)              ;; BUG: always 0 or 1

;; Solution: unique-by AgentLink means fold sees one ApiCall per agent,
;; so 5 agents with ApiCall=1 correctly sums to 5.
(with-unique-by [AgentLink]
  (fold 0 ApiCall + ApiCount))           ;; CORRECT: sums per agent
```

This commonly arises after `with-table-if` merges branches where different rows share the same value for the fold symbol:

```oql
;; Each order is either processed (Cost = actual cost) or skipped (Cost = 0)
(with-table-if (should-process OrderId)
  [OrderId Cost]
  (= Cost OrderTotal)
  (= Cost 0))

;; Default fold would only see unique values of Cost
;; (e.g., 0, 99, 149 — collapsing multiple $99 orders into one)
;; with-unique-by [OrderId] preserves per-order granularity
(with-unique-by [OrderId]
  (fold 0 Cost + TotalCost))
```

`with-unique-by` wraps a body — any `fold` inside that body uses the unique-by constraint:

## with-order-by

Sorts solutions before folding or returning:

```oql
(with-order-by [Score]
  (fold [] Name append SortedNames))     ;; Names sorted by Score ascending
```

## with-group-by

Partitions solutions into groups. Inside the body, `fold` operates per-group:

```oql
(with-group-by [Department]
  (fold 0 Salary + DeptTotal))           ;; DeptTotal per department
```

## with-count

Counts solutions matching a term:

```oql
(with-count Total (Qo.Db.Prop/prop-vals FolderId _ _))  ;; Total = row count
```

## Performance: partitions and `run-term`

`fold` / `with-group-by` / `with-order-by` scale poorly when the solution table contains many distinct group-key values ("partitions"). Per-partition bookkeeping compounds non-linearly with partition count, so a query that iterates N keys and folds per key can be orders of magnitude slower than the underlying term read would suggest.

The idiomatic workaround is to **establish the partition boundary before folding**: wrap the reducer body in a clause and invoke it once per key via `run-term`. Each invocation's solution is single-partition, so the reducer chain runs against only that partition's rows.

Measured 2026-04-24 for a 21-page pull: 8.3s with a top-level `with-group-by [PageId]`, 0.5s when the same reducer body was called via `run-term` per page. See [query-omega-oql/docs/explanation/fold-partition-scaling.md](../../query-omega-oql/docs/explanation/fold-partition-scaling.md) for the full explanation and canonical SLOW/FAST pattern.

## fold pitfalls

### fold is solution-wide, not per-row

`fold` is a solution-wide aggregation — think SQL `GROUP BY` the whole table. It collapses all rows into one accumulated value. When you write `(fold [] P append Fields)`, fold walks every value `P` resolves to across the entire solution and appends each one into the same accumulator. The result is a single shared array that gets bound to `Fields` on every row.

**Symptom:** An enrichment step uses `(fold [] P append Fields)` inside a `with-table-if` branch to build one array per row. At batch=1 (single row) it works by accident. At batch>1, every row gets the same merged array containing ALL rows' values.

**The fix — use `append`:**

`append` is a per-row operation (like Clojure's `conj`). It adds an element to a list within the current row's bindings, not across the solution.

```oql
;; BAD — fold builds ONE shared array across all rows
(with-table-if [ContactId FieldPair] (defined FieldPair)
  (fold [] FieldPair append Fields))
;; batch=2 -> Fields = [pair-A pair-B] (both contacts get BOTH pairs)

;; GOOD — append builds one array per row
(with-table-if [ContactId FieldPair] (defined FieldPair)
  (append Fields FieldPair Fields1))
;; batch=2 -> contact A gets [pair-A], contact B gets [pair-B]
```

| Operation | Scope | Analogy | Use for |
|-----------|-------|---------|---------|
| `fold` | solution-wide | SQL `GROUP BY *` / Clojure `reduce` over the whole table | totals, counts, collecting all distinct values into one result |
| `append` | per-row | Clojure `conj` on the current row | building an array that belongs to each row independently |

### fold folds over unique values, not rows

`fold` folds over the *unique values* of its input symbol, not over rows. If `X` is a column whose values are `0` or `1`, `fold` sees two unique values (`0` and `1`) and returns a sum of 1, regardless of how many rows had `X = 1`.

**Symptom:** `(fold 0 X + Sum)` returns a value that maxes out at 1 (or at the count of distinct values of `X`), not the expected row count.

**The fix:** Use `with-unique-by` to make fold see one value per unique row key:

```oql
(Qo.Db.Prop/prop-val! "FolderId" "PageId" "NeedsCreate" NeedsCreate)

;; BAD — Sum maxes at 1 even if 500 rows have NeedsCreate
(fold 0 NeedsCreate + Sum)

;; GOOD — Sum = count of rows
(with-unique-by [RowId]
  (fold 0 NeedsCreate + Sum))
```

Or count rows directly with a literal `1` instead of a column:

```oql
(with-table-if [RowId NeedsCreate] (= NeedsCreate "true")
  (fold 0 1 + Count))
```

### Why batch=1 masks fold bugs

With one row in the solution, "all rows" and "this row" are the same set. The fold accumulator contains exactly one row's values, so the output looks correct. Both fold pitfalls only surface when the batch contains two or more rows.

## with-limit

Limits the number of solutions returned. Threads the limit into CouchDB view queries:

```oql
;; Return only the first 10 pages
(with-limit 10
  (Qo.Db.Prop/prop-vals FolderId PageId PropName PropVal))
```

`with-limit` wraps a body — terms inside the body respect the limit. Combine with `with-skip` for pagination:

```oql
(with-skip 20
  (with-limit 10
    (Qo.Db.Prop/prop-vals FolderId PageId PropName PropVal)))
```

## with-skip

Skips the first N results from a view-indexed query. Same scoping pattern as `with-limit`:

```oql
(with-skip 20
  (with-limit 10
    (Qo.Db.Prop/prop-vals FolderId PageId PropName PropVal)))
```

## with-sort

Controls the sort direction of view-indexed query results. Threads `descending: true` (or the default ascending) into the CouchDB view request:

```oql
;; Fetch rows in descending order
(with-sort "desc"
  (Qo.Db.Prop/prop-vals FolderId PageId PropVal))

;; Combine with limit to get the last N rows
(with-sort "desc"
  (with-limit 5
    (Qo.Db.Prop/prop-vals FolderId PageId PropVal)))
```

Values:
- `"asc"` — ascending order (default; no-op since views return ascending by default)
- `"desc"` — descending order (sets `descending: true` on the view query)

Same ctx-scoping pattern as `with-limit` and `with-skip` — wraps a body, only affects view-indexed queries inside that body.
