[< Language](../language.md) | [OmegaDB](../../README.md#omegadb)

# Solution Table Operations (Reducers)

These operations work on the solution table — they aggregate, filter, or reorder rows across the solution set.

```oql
(fold Init Sym Func Agg)                  ;; reduce across solutions
(with-count Count Term)                   ;; count solutions matching a term
(with-order-by [Sym] Body)               ;; order solutions by symbol value
(with-group-by [Sym] Body)               ;; group solutions by symbol value
(with-unique-by [Sym] Body)              ;; deduplicate solutions by symbol value
(with-limit N Body)                       ;; limit to first N solutions
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

## with-limit (not yet implemented)

**Warning:** `with-limit` is parsed by the interpreter but not fully implemented in the runtime. It will crash if used. This is a known TODO.
