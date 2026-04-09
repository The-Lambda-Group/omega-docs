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

**Each branch is its own lexical scope**, just like a clause body. The header `[Var1 Var2 Result]` acts like a clause signature — it defines what goes in and what comes out. Variables created inside a branch are local to that branch and don't need to be in the header. Only variables that the outer scope needs to read after the `with-table-if` must appear in the header.

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

When the solution table has multiple rows entering a `with-table-if`, each row independently evaluates the condition and enters the appropriate branch. The results from both branches are merged back into a single table (like a SQL `UNION` of the two branches):

```oql
;; Given a list of items with mixed types
(get Items _ Item)
(get Item "type" Type)
(get Item "value" Value)

;; Each item goes through its own branch based on type
(with-table-if (= Type "number")
  [Value Output]
  (* Value 100 Output)                    ;; then: scale numbers
  (and (string-concat "label:" Value S)   ;; else: format strings
       (= Output S)))                     ;; S is local to this branch

;; Output is bound per-item — the solution set has one row per item
(fold [] Output append Results)
```

**Nesting** is allowed for multi-step parsing but prefer flat `when` blocks or separate clauses when possible:

```oql
;; Nested with-table-if for parsing (acceptable pattern)
(with-table-if (= RawSales "")
  [RawSales Sales]
  (= Sales 0)
  (with-table-if (number RawSales)
    [RawSales Sales]
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
