[< Language](../language.md) | [OmegaDB](../../README.md#omegadb)

# Control Flow

OQL's control flow operates on solution tables. Understanding the distinction between universal quantifiers (`when`, `when-not`, `if`) and per-solution branching (`with-table-if`) is critical.

## when / when-not / if (universal quantifiers)

These operate as **"if any"** — they pass or fail based on whether *any* solution in the current solution set satisfies the condition. They don't branch per-solution.

```oql
;; (when) — "when any": runs body if the condition succeeds for any solution
(when (= Step "score")
  (= PageName "Score Agents"))

;; (when-not) — "when no": runs body if the condition fails for all solutions
(when-not (get Data "skip" Skip)
  (= Skip 0))

;; (if) — "if any": binds one of two values based on whether condition succeeds
(if (get Data "continue" "false")
  (= ShouldContinue "false")
  (= ShouldContinue "true"))
```

Use `when` for flat dispatch (like a switch/case):
```oql
(when (= Step "score")  (= PageName "Score Agents"))
(when (= Step "filter") (= PageName "Filter Agents"))
(when (= Step "batch")  (= PageName "GHL Sync"))
```

## with-table-if (per-solution branching)

`with-table-if` is fundamentally different — it branches **per solution**. The condition evaluates within the parent solution, and each branch creates its own table solution.

```oql
(with-table-if Condition
  [Var1 Var2 Result]          ;; table head — variables in scope for both branches
  ThenBranch                   ;; runs for solutions where Condition succeeds
  ElseBranch)                  ;; runs for solutions where Condition fails
```

### SQL analogy

If you come from SQL, think of `with-table-if` as four steps:

```sql
-- 1. Project the parent solution down to the schema. Duplicate rows
--    (rows that match on every schema column) collapse here. This is
--    the step that causes silent row loss if the schema doesn't
--    include a unique per-row identifier.
WITH projected AS (
  SELECT DISTINCT <schema> FROM parent_solution
),

-- 2. Then-branch: rows where <condition> holds, INNER JOIN'd against
--    whatever new bindings the then-branch body produces. If the body
--    fails (e.g. `(= true false)`), the join has no match → row drops.
--    This is the mechanism behind the "drop a row from the solution"
--    idiom.
then_rows AS (
  SELECT projected.*, then_body.*
  FROM projected
  INNER JOIN <then_branch_body> then_body
    ON <condition>
),

-- 3. Else-branch: rows where <condition> fails, LEFT OUTER JOIN'd
--    against the else-branch body. The conventional else-branch is
--    `(= true true)` — a no-op pass-through — so the left join adds
--    nothing and the row just survives unchanged.
else_rows AS (
  SELECT projected.*, else_body.*
  FROM projected
  LEFT OUTER JOIN <else_branch_body> else_body
    ON NOT <condition>
),

-- 4. Union the two halves back into the output solution.
output AS (
  SELECT * FROM then_rows
  UNION
  SELECT * FROM else_rows
)
```

Three things fall out of this model that match what you'll see in practice:

1. **The `SELECT DISTINCT` at step 1 is where row collapse happens.** If the schema isn't wide enough to uniquely identify each parent row, the distinct projection merges duplicates — and all the per-row state that isn't in the schema is lost with them. This is why `with-table-if` schemas should always include a row-identity column (like a primary key or URL) even if neither branch body reads it. See "Preserving per-row identity" below.

2. **The then-branch is an inner join.** `(= true false)` in the then-branch fails unification, which fails the join on that row — the row drops from `then_rows` and never makes it into the final union. That's the idiom for "filter this row out." See "Dropping rows from the solution" below.

3. **The else-branch is conventionally a left outer join.** Almost every else-branch is `(= true true)` — a no-op pass-through. Rows take the else branch, bind nothing new, continue forward. You *can* make the else-branch fail to drop rows there too, but that's unusual — when you want "drop these rows, keep the rest," the idiomatic pattern is to put the "drop" condition in the then-branch with `(= true false)` and leave the else-branch as `(= true true)`.

### Conditionally binding variables

The table head `[Var1 Var2 Result]` declares which symbols flow into both branches. This is how you **conditionally bind variables**:

```oql
;; Not every Data has "key" — conditionally bind Val
(with-table-if (get Data "key" Val)
  [Data Val Result]
  (= Result {"found" Val})           ;; then: Val is bound
  (= Result {"found" "default"}))    ;; else: Val was not in Data
```

Inside each branch, you can define new symbols with `(and ...)` — they get wrapped into the table:

```oql
(with-table-if (= RawSales "")
  [RawSales Sales]
  (= Sales 0)                        ;; then: empty string -> 0
  (and (string-replace RawSales "," "" Clean)    ;; else: parse it
       (json-stringify Sales Clean)))             ;; new symbols OK inside (and)
```

Both branches evaluate and the results are combined — the solution set continues with all rows from both branches.

**Each branch is its own lexical scope**, just like a clause body. The header `[Var1 Var2 Result]` acts like a clause signature — it defines what goes in and what comes out. Variables created inside a branch are local to that branch and don't need to be in the header. Variables that the outer scope needs to read after the `with-table-if` must appear in the header — but read the "Preserving per-row identity" section below before deciding what's "enough" to include.

```oql
;; Parse a value that might be empty, numeric, or a formatted string
;; Each branch has its own local variables — Clean is local to the else branch
(with-table-if (= RawValue "")
  [RawValue Parsed]
  (= Parsed 0)
  (and (string-replace RawValue "," "" Clean)
       (json-stringify Parsed Clean)))

;; Clean does not exist here — only RawValue and Parsed carry forward
(= Result {"value" Parsed})
```

### Preserving per-row identity

**This is the most common source of silent row loss in `with-table-if` code, and it has a specific shape you need to learn.**

When the solution table has multiple rows entering a `with-table-if`, each row independently evaluates the condition and enters the appropriate branch. The results from both branches are merged back into a single table.

The merge is effectively a `SELECT DISTINCT <schema>` union. **If two rows in the parent solution have identical values for every variable in the schema, the merge collapses them into one row** — even though logically they represent different items. Everything not in the schema that was per-row is lost.

Always include **at least one variable that is unique per row** in the schema — typically a primary key or natural key like an ID, URL, or UUID. The condition and branch logic will still run per-row (that part isn't what breaks), but the union merge needs the row-identity column to keep the rows distinct.

```oql
;; BAD — silent row collapse
;; If two agents happen to have the same AgentStatus and the same HttpStatus,
;; they merge into one row and one of them disappears.
(with-table-if (and (= AgentStatus "create")
                    (= HttpStatus 422))
  [AgentStatus HttpStatus]     ;; ← schema has no per-row identity
  (= true false)                ;; drop invalid rows
  (= true true))

;; GOOD — AgentLink anchors per-row identity
(with-table-if (and (= AgentStatus "create")
                    (= HttpStatus 422))
  [AgentLink AgentStatus HttpStatus]   ;; ← AgentLink is unique per agent
  (= true false)
  (= true true))
```

The "it works in my test with 1 or 2 rows" trap is especially insidious because a 2-row test almost never collides on a narrow schema — you have to run at scale to see the collapse. If you see unbound symbols surfacing downstream of a `with-table-if`, or row counts dropping below what you expect, suspect a missing identity column first.

**Rule of thumb:** if your `with-table-if` runs over a per-row solution table, the schema should always include the column(s) you'd use as a primary key. If you're not sure whether a row is per-row or global, err on including more.

### Dropping rows from the solution

A `with-table-if` can be used to **drop** rows from the downstream solution by having the then-branch fail to unify. The idiomatic form is `(= true false)`:

```oql
;; Drop every row where GHL returned a 422. These rows are unrecoverable
;; and we don't want them reaching the write step with unbound values.
(with-table-if (and (= AgentStatus "create")
                    (= HttpStatus 422))
  [AgentLink AgentStatus HttpStatus]
  (= true false)    ;; then: fail the branch → row drops from the solution
  (= true true))    ;; else: pass through unchanged
```

`(= true false)` unifies two values that cannot match, which fails the branch. In the `UNION` analogy, this is equivalent to `SELECT ... WHERE 1=0` for one of the two CTEs — the rows matching the condition simply don't appear in the output.

### When the condition is batch-wide, not per-row

Several terms that look per-row actually evaluate **once against the whole parent solution**, returning true/false globally. If you gate a `with-table-if` on one of these, every row takes the same branch — which is almost never what you want for per-row dispatch.

The term to watch out for is `(defined X)`:

```oql
;; BAD — (defined X) is effectively universal. If ANY row in the solution
;; has SuccessId bound, EVERY row takes the then-branch, even rows that
;; didn't go through the path that bound SuccessId.
(with-table-if (defined SuccessId)
  [AgentLink SuccessId GhlContactId AgentStatus]
  (and (= GhlContactId SuccessId)
       (= AgentStatus "created"))
  (= true true))
```

**Always combine `(defined X)` with a concrete per-row predicate** that narrows the branch to the rows you actually want:

```oql
;; GOOD — (= AgentStatus "create") is per-row, so rows in "skipped" state
;; never enter the then-branch even if SuccessId happens to be bound
;; somewhere else in the solution.
(with-table-if (and (= AgentStatus "create")
                    (defined SuccessId))
  [AgentLink SuccessId GhlContactId AgentStatus]
  (and (= GhlContactId SuccessId)
       (= AgentStatus "created"))
  (= true true))
```

Better: skip `(defined X)` entirely when you can. Use a concrete value check like `(= HttpStatus 200)` or `(get Row "field" Val)` (which fails per-row if the field is missing).

### Nesting

**Nesting is allowed** for multi-step parsing but adds real costs:

- **Stack traces lose the branch boundary.** When something fails inside a nested `with-table-if`, the trace dumps the entire outer branch body, making it hard to tell which branch was actually executing. Debugging nested failures is significantly harder than debugging flat sequential `with-table-if`s.
- **Schema discipline compounds.** Inner `with-table-if`s need all the row-identity concerns of outer ones plus any state the outer branch just bound.

Prefer **flat sequential `with-table-if`s** over nesting when possible — each one handles one piece of classification, the next one reads the result of the previous via normal per-row unification:

```oql
;; GOOD — flat, sequential. Each step does one thing.
(with-table-if (= RawSales "")
  [AgentLink RawSales SalesBucket]
  (= SalesBucket "empty")
  (= SalesBucket "nonempty"))

(with-table-if (and (= SalesBucket "nonempty")
                    (number RawSales))
  [AgentLink RawSales SalesBucket Sales]
  (= Sales RawSales)
  (= true true))

(with-table-if (and (= SalesBucket "nonempty")
                    (not (number RawSales)))
  [AgentLink RawSales SalesBucket Sales]
  (and (string-replace RawSales "," "" Clean)
       (json-stringify Sales Clean))
  (= true true))
```

When you do nest, keep the inner `with-table-if`'s schema at least as wide as the outer's for the per-row identity column(s):

```oql
;; Acceptable nesting — both levels carry AgentLink
(with-table-if (= RawSales "")
  [AgentLink RawSales Sales]
  (= Sales 0)
  (with-table-if (number RawSales)
    [AgentLink RawSales Sales]
    (= Sales RawSales)
    (and (string-replace RawSales "," "" Clean)
         (json-stringify Sales Clean))))
```

## When to use which

| Pattern | Use when |
|---------|----------|
| `when` | Flat dispatch — run body if condition matches any solution |
| `when-not` | Default values — bind a symbol when it's not already bound |
| `if` | Bind one of two values based on a condition |
| `with-table-if` | Per-solution branching — conditionally bind variables, parse values |
