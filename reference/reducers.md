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

Counts solutions matching a term. Always produces exactly one solution row binding the count — including `0` when the term matches nothing. This makes it the building block for empty-handling around `fold` (see "fold over zero solutions does not bind its accumulator" below).

```oql
(with-count Total (Qo.Db.Prop/prop-vals FolderId _ _))  ;; Total = row count
```

**Restrict the inner term to a single call.** `with-count` over a compound condition — `(with-count Cnt (and Term1 Term2))` — currently fails with a `NullPointerException`. If you need to count rows matching a compound predicate, wrap the predicate in an in-memory clause and count over the clause invocation. See the "fold over zero solutions" section below for the full pattern.

**The "wrap in a clause" workaround is insufficient when the wrapping clause contains a `prop-vals-by-folder` scan** — that variant also NPEs. Three distinct `with-count` NPE patterns are documented with a safe universal workaround in [gotchas/with-count-captured-helper-npe.md](../gotchas/with-count-captured-helper-npe.md). For any code that counts rows via a captured helper or compound predicate involving `prop-vals-by-folder`, use the `with-table-if` + `fold`-over-per-row-unique-symbol + `length` pattern documented there.

## Performance: partitions and `run-term`

`fold` / `with-group-by` / `with-order-by` scale poorly when the solution table contains many distinct group-key values ("partitions"). Per-partition bookkeeping compounds non-linearly with partition count, so a query that iterates N keys and folds per key can be orders of magnitude slower than the underlying term read would suggest.

The idiomatic workaround is to **establish the partition boundary before folding**: wrap the reducer body in a clause and invoke it once per key via `run-term`. Each invocation's solution is single-partition, so the reducer chain runs against only that partition's rows.

Measured 2026-04-24 for a 21-page pull: 8.3s with a top-level `with-group-by [PageId]`, 0.5s when the same reducer body was called via `run-term` per page. See [query-omega-oql/docs/explanation/fold-partition-scaling.md](../../query-omega-oql/docs/explanation/fold-partition-scaling.md) for the full explanation and canonical SLOW/FAST pattern.

**There are two valid `run-term` shapes for captured in-memory helpers:**

1. **Partition-before-fold** (this section) — replace `with-group-by` with a per-partition `run-term` invocation. Each invocation's solution is single-partition, so reducers run against only that partition's rows.
2. **Per-row iteration at scale** — invoke a helper once per row in a multi-row solution table where K ≥ ~30 rows. Empirically (MAB Allocate V1, K=40 arms, 2026-04-30), `call` per-row hangs at this scale; `run-term` per-row drops wall-clock to 3–5s. The fresh-solution isolation mechanism is the same as in partition-before-fold. See [run-term per-row in multi-solution iteration](../explanation/run-term-per-row-iteration.md) for the full pattern and empirical data.

Outside these two shapes — for one-solution-in / one-solution-out helpers, list-mapping helpers, validation helpers, etc. — use `call`. `run-term` against a captured in-memory helper in any other context produces a 500 or engine spin-out / `ECONNRESET`. See [run-term does not work on in-memory clauses](../gotchas/run-term-in-memory-clause.md) for the rule statement and the captured-helper framing.

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

### Counting rows in a captured-helper body when with-count is unavailable

When `with-count` is unavailable — due to any of its known NPE patterns (compound `(and ...)` predicate, sec-index trailing-wildcard, or a captured helper containing `prop-vals-by-folder`) — use the **fold-append-per-row-unique-symbol** pattern to count matching rows:

```oql
(:- (CountRowsMatchingPredicate ScanArgs FilterArgs Count)
    (with-table-if (and (<scan term producing per-row symbols>)
                        (<filter terms>))
      [<header symbols including PerRowUniqueSymbol and RowList>]
      (fold [] PerRowUniqueSymbol append RowList)
      (= RowList []))
    (length RowList Count))
```

**The load-bearing detail:** fold must be over a *per-row-unique* symbol, not a constant or a low-cardinality column. Since fold folds over unique values (see "fold folds over unique values, not rows" above), folding a symbol that has one distinct value per matching row produces one list entry per row — no dedup loss.

In a CouchDB-backed `prop-vals-by-folder` scan, the page-id (`Pid` / `SaPid` etc.) is a reliable per-row-unique symbol because each CouchDB document has a unique `_id`.

**Concrete example** (MAB Allocate V1, `CountSaForArm`, commit `5341753`):

```oql
(:- (CountSaForArm SaTableId Rev ExpId SaCount)
    (with-table-if (and (Qo.Public.OqlApi.Db.Prop/prop-vals-by-folder SaTableId SaPid SaPv)
                        (get-in SaPv ["prop-vals" "revision"] Rev)
                        (get-in SaPv ["prop-vals" "experiment_id"] ExpId))
      [SaTableId Rev ExpId SaPid SaPv SaList]
      (fold [] SaPid append SaList)
      (= SaList []))
    (length SaList SaCount))
```

**Why each piece is required:**

| Piece | Role |
|---|---|
| `with-table-if` condition | Upstream-cardinality gate (canonical fold-with-default idiom). If zero rows match, the else-branch fires. |
| else-branch `(= RowList [])` | Binds the accumulator to `[]` on the empty path, so `length` sees a bound symbol on both paths. |
| `fold [] SaPid append SaList` | Folds the per-row-unique page-id. One unique `SaPid` per CouchDB row → one list entry per matching row. |
| `(length SaList SaCount)` OUTSIDE `with-table-if` | Runs against whichever branch's binding survived. Produces `Count = 0` on the empty path. |

**Why `(fold 0 1 + Count)` fails in a captured-helper body:** the literal-1 approach (documented above) only works when the surrounding solution context already has per-row distinctness. Inside a captured-helper body whose scan is `prop-vals-by-folder`, the implicit cardinality is governed by the helper's argument bindings, not by per-row identity. `1` is a constant — fold sees a single unique value and returns `1` regardless of how many rows the scan produced. Use a per-row-unique symbol instead.

**When to use this pattern:** any time you need a row count inside a captured-helper body and any of the `with-count` NPE patterns applies. If `with-count` works in your context (single-clause predicate, no captured helper containing `prop-vals-by-folder`), the canonical fold-with-default idiom (§ "fold over zero solutions") is simpler. See [gotchas/with-count-captured-helper-npe.md](../gotchas/with-count-captured-helper-npe.md) for the full list of `with-count` NPE patterns and their workarounds.

### Counting source-array length vs. fold-accumulator length

The same "fold sees unique values, not rows" rule has a second consequence: a uniqueness-validation check that compares `(length AllKeys ...)` against `(length UniqueKeys ...)` after both have been produced by `fold` will always report "no duplicates" — because both arrays are already deduplicated by the time `length` measures them.

If the goal is to detect duplicate keys in an LLM-produced (or user-produced) array of records before any write, **measure the source array directly with `length`, not the post-fold accumulator**:

```oql
;; WRONG — Total counts unique keys, not records. Duplicates always pass.
(get Drafts _ Draft)
(get Draft "treatment_id" Tid)
(get Draft "name" Name)
(string-concat Tid "::" Partial)
(string-concat Partial Name PairKey)
(fold [] PairKey append AllKeys)
(length AllKeys Total)               ;; ← already-deduplicated count
(call IntoSet AllKeys UniqueSet)
;; ...comparison is tautological.

;; RIGHT — Total = source-array length, before fold dedupes.
(get Drafts _ Draft)
(get Draft "treatment_id" Tid)
(get Draft "name" Name)
(string-concat Tid "::" Partial)
(string-concat Partial Name PairKey)
(fold [] PairKey append AllKeys)
(length Drafts Total)                ;; ← measure the input array
(call IntoSet AllKeys UniqueSet)
(get UniqueSet UKey _)
(fold [] UKey append UniqueKeys)
(length UniqueKeys UniqueCount)
(with-table-if (= UniqueCount Total) [...] ok throw-duplicate-keys)
```

The exemplar in MAB Manager's `ValidateCommands` is the same shape — `(length Commands TotalCount)` against `(length UniqueAgents UniqueCount)`. The rule generalises: any "are there duplicates in this array?" check must measure the source array's length, not a folded view of it. See [explanation/agent-step-write-shapes.md § Pattern 1](../explanation/agent-step-write-shapes.md#pattern-1--uniqueness-validation-against-a-source-array-not-a-folded-accumulator) for the agent-step-level discussion.

### Why batch=1 masks fold bugs

With one row in the solution, "all rows" and "this row" are the same set. The fold accumulator contains exactly one row's values, so the output looks correct. Both fold pitfalls only surface when the batch contains two or more rows.

### fold over zero solutions does not bind its accumulator

`fold` only produces an output binding when its upstream term produces at least one solution. If the upstream has zero solutions, `(fold Init Sym Func Agg)` does **not** bind `Agg` to `Init` — it does not bind `Agg` at all. The `Init` value is the *seed for the reduction*, not a default for the empty case.

This is the most common cause of a downstream `omega/query/no-return` error in queries that look correct on the happy path:

```oql
;; If no pages match the sec-index lookup, AllPids is unbound,
;; so the (return [AllPids]) at the bottom of the query has nothing to return.
(Qo.Public.OqlApi.Db.Prop/sec-index FolderId "agent" "researcher" Pid)
(fold [] Pid append AllPids)
(return [AllPids])
```

The fold succeeds silently in the sense that no error fires *at* the fold — the failure manifests further down the query when something tries to read the unbound `Agg` symbol. The error surface point misleads agents into checking the `(return ...)` instead of the upstream cardinality.

#### The canonical fold-with-default idiom

Use `with-count` to materialize the upstream cardinality into a single-row solution, then branch on the count with `with-table-if`. The then-branch runs the real query and folds; the else-branch binds the accumulator to the empty default explicitly:

```oql
;; Want: AllPids = list of matching Pids, or [] if none match.
(with-count Cnt (Qo.Public.OqlApi.Db.Prop/sec-index FolderId "agent" "researcher" _))
(with-table-if (> Cnt 0)
  [FolderId AllPids]
  (and (Qo.Public.OqlApi.Db.Prop/sec-index FolderId "agent" "researcher" Pid)
       (fold [] Pid append AllPids))
  (= AllPids []))
;; AllPids is now bound on every downstream row, regardless of cardinality.
```

Why this works:

- `with-count` always produces exactly one solution — even when the inner term matches nothing — and binds `Cnt` (to `0` in the empty case).
- That one-row solution is what flows into `with-table-if`. Both branches have a row to operate on.
- The then-branch re-runs the upstream term as a real reader, so `fold` sees the actual N solutions and reduces them.
- The else-branch never runs the term — it just binds `AllPids` to the seed value directly.

The schema header on `with-table-if` must include `AllPids` (so it survives the merge) and a per-row identity column from the parent solution (here `FolderId` — see "Preserving per-row identity" in [control-flow.md](control-flow.md) for the full rule). Schemas missing the identity column collapse rows in the multi-row caller case.

#### Why the obvious workarounds don't work

- **Naked `with-table-if (sec-index ...)`** — the condition runs per parent row, but if the parent solution itself has zero rows entering, the with-table-if has nothing to branch on and nothing comes out. Use `with-count` first so there is always exactly one row to branch.
- **`(if (> Cnt 0) ...)`** — `if` is a universal quantifier (see [control-flow.md](control-flow.md)), not a per-row branch, and it does not bind variables across both arms. Use `with-table-if`.
- **`with-count` over a compound condition** — `(with-count Cnt (and Term1 Term2))` currently fails with a `NullPointerException`. Restrict the term inside `with-count` to a single call. If you need to count rows matching a compound predicate, push the compound logic into a `(:- ...)` in-memory clause and invoke that clause inside `with-count`:

    ```oql
    ;; BAD — NPE
    (with-count Cnt (and (sec-index F "k" "v" Pid)
                         (Qo.Page/page-object _ _ Pid _)))

    ;; GOOD — wrap the compound predicate in an in-memory clause
    (:- (Match F Pid)
        (and (Qo.Public.OqlApi.Db.Prop/sec-index F "k" "v" Pid)
             (Qo.Page/page-object _ _ Pid _)))
    (with-count Cnt (Match FolderId _))
    ```
- **Counting all rows in the folder, then gating** — works if the folder is entirely empty but does not help when the folder has rows for *other* values that get filtered out by the lookup. The count must measure the same predicate the fold will reduce over.

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
