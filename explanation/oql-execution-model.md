[< Home](../README.md) | [Explanation](README.md)

# OQL Execution Model

Understanding how OQL executes is essential for writing correct and efficient code. OQL is not procedural — it's a logic language where every statement is a constraint, and execution is the process of finding all consistent solutions.

## Solution Sets

The fundamental data structure in OQL is the **solution set**. A solution set is a table of bindings — each row is a possible assignment of values to symbols.

```oql
;; After this, the solution set has one row: {X: 5}
(= X 5)

;; After this, still one row: {X: 5, Y: 6}
(+ X 1 Y)
```

When a term produces multiple results, the solution set grows:

```oql
;; If the datastore has 3 pages, the solution set has 3 rows
(Qo.Page/page-object _ FolderId _ Page)
;; Row 1: {FolderId: "abc", Page: {...}}
;; Row 2: {FolderId: "abc", Page: {...}}
;; Row 3: {FolderId: "abc", Page: {...}}
```

Every subsequent term is **joined** against this table — like a SQL JOIN. This is why OQL can blow up: if you have 1000 rows from a page query and then do a prop-vals lookup, you're doing a 1000-way join.

## Terms as Joins

Think of each term as a CTE (Common Table Expression) in SQL. The engine:

1. Takes the current solution set
2. Evaluates the term for each row
3. Joins the results back into the solution set

```oql
;; Step 1: 1000 rows (one per page)
(Qo.Public.OqlApi.Page/page-query Q Page)

;; Step 2: for each of the 1000 rows, get page-id → still 1000 rows
(get Page "page-id" RowPageId)

;; Step 3: for each of the 1000 rows, look up prop-vals → still 1000 rows
;; BUT the solution set now carries Page, RowPageId, AND PropVals per row
(Qo.Db.Prop/prop-vals FolderId RowPageId PropVals)
```

The solution set accumulates bindings. By the time you reach the end of a clause, every symbol from every term is in the table. This is where memory pressure comes from.

## Clauses as Solution Boundaries

A clause call acts as a **solution boundary**. When you call a clause:

1. The current solution is narrowed to just the head arguments (input)
2. The clause body executes with a fresh solution
3. Only the head arguments (output) come back

This is why `def-query` and `(:- ...)` clauses are the primary tool for keeping solutions small:

```oql
;; BAD: all intermediate symbols accumulate in the solution
(Qo.Db.Prop/prop-vals FolderId RowPageId PropVals)
(get PropVals "prop-vals" PV)
(get PV "agent_profile_link" Link)
(get PV "agent_full_name" Name)
(get PV "closed_sales" RawSales)

;; GOOD: def-query swallows intermediate symbols
(def-query GetFieldsQ [FolderId RowPageId Link Name RawSales]
  (Qo.Db.Prop/prop-vals FolderId RowPageId PropVals)
  (get PropVals "prop-vals" PV)
  (get PV "agent_profile_link" Link)
  (get PV "agent_full_name" Name)
  (get PV "closed_sales" RawSales))

;; Only FolderId, RowPageId, Link, Name, RawSales exist in the outer solution
(with-query GetFieldsQ)
```

## Clause Types

### Stored Clauses (`(:- (Ns/name! ...) ...)`)

Written to a datastore. The `!` suffix means "store this clause." Available for future queries to call.

```oql
;; This clause is stored in the Qo.Page datastore
(:- (Qo.Page/visible! PageId)
    (Qo.Page/page-object _ ParentId PageId _)
    (Qo.Page.Bl/page-block _ ParentId _ Block)
    (get Block "page-id" PageId))
```

### In-Memory Clauses (`(:- (Name ...) ...)`)

No namespace prefix, no `!`. Lives only in the current query execution. Used for helper logic within an implementation.

```oql
;; Exists only during this query — not stored anywhere
(:- (ParseSales RawSales Sales)
    (with-table-if (= RawSales "")
      [RawSales Sales]
      (= Sales 0)
      (json-stringify Sales RawSales)))
```

### def-query

Defines an inline query within a clause body. Acts as a solution boundary — intermediate symbols don't leak out.

```oql
(def-query MyQ [InputVar OutputVar]
  ;; These symbols are scoped to this def-query
  (some-term InputVar Intermediate)
  (other-term Intermediate OutputVar))

;; Bring results into the outer solution
(with-query MyQ)
;; Only InputVar and OutputVar are in scope here
```

## Control Flow Reference

### when / when-not

Conditional execution. If the condition fails, the body is skipped (no error).

```oql
;; Run body only if Step is "score"
(when (= Step "score")
  (= PageName "Score Agents"))

;; Run body only if Data doesn't have "skip"
(when-not (get Data "skip" Skip)
  (= Skip 0))

;; Multiple when blocks for flat dispatch (like a switch/case)
(when (= Step "score")  (= PageName "Score Agents"))
(when (= Step "filter") (= PageName "Filter Agents"))
(when (= Step "batch")  (= PageName "GHL Sync"))
```

### with-table-if

Conditional branching with explicit variable scope. Use when you need to bind different values depending on a condition.

```oql
(with-table-if (get Data "key" Val)
  [Data Val Result]              ;; variables that flow into both branches
  (= Result "found")            ;; then: condition succeeded
  (= Result "not found"))       ;; else: condition failed
```

The variable list `[Data Val Result]` declares which symbols are in scope for both branches. This is required because OQL needs to know which bindings to carry forward.

**Nesting is allowed but discouraged** — prefer flat `when` blocks or separate clauses:

```oql
;; Parsing with nested with-table-if (acceptable for value parsing)
(with-table-if (= RawSales "")
  [RawSales Sales]
  (= Sales 0)
  (with-table-if (number RawSales)
    [RawSales Sales]
    (= Sales RawSales)
    (and (string-replace RawSales "," "" Clean)
         (json-stringify Sales Clean))))
```

### and / or

Logical connectives. `and` requires all terms to succeed. `or` succeeds if any branch succeeds (creates multiple solution paths).

```oql
;; All must succeed
(and (get Data "name" Name)
     (get Data "email" Email)
     (= Result {"name" Name "email" Email}))

;; Any branch can succeed — CAREFUL: creates multiple solutions
(or (and (string-contains Value "B") (* Num 1000000000 Result))
    (and (string-contains Value "M") (* Num 1000000 Result))
    (and (string-contains Value "K") (* Num 1000 Result)))
```

### fold

Collects multiple solutions into a single list. This is how you go from "one row per solution" to "one list of all rows."

```oql
;; Row is bound once per solution (e.g., 1000 times for 1000 rows)
;; fold collects all Row values into Rows
(fold [] Row append Rows)
```

### with-count

Counts the number of solutions for a term without materializing them all:

```oql
(with-count TotalCount (PropValsDs/property-values FolderId _ _))
```

### with-order-by

Orders solutions by specified symbols before collecting:

```oql
(with-order-by [Score]
  (fold [] Row append SortedRows))
```

## Index Matching

Due to engine limitations, term queries against a datastore must match an index or primary key **exactly** — binding the correct arguments in the correct positions. If the query doesn't match an index, the engine falls back to a **full table scan**, which can crash the database on large tables.

Every `defterm` declares a primary key. Indexes are defined on specific argument positions (e.g., `["args.0"]`, `["args.0", "args.1"]`). A query matches an index only if the bound arguments align exactly with the index fields.

```oql
;; defterm: page-object with args [AppId, FolderId, PageId, Page]
;; Index on: ["args.2"] (PageId)

;; GOOD — binds PageId (args.2), matches the index
(Qo.Page/page-object _ _ PageId Page)

;; GOOD — binds FolderId (args.1), matches a different index
(Qo.Page/page-object _ FolderId _ Page)

;; BAD — binds AppId AND FolderId, no index covers both → full table scan
(Qo.Page/page-object AppId FolderId _ Page)
```

**Rules:**
- Check the defterm's primary key and indexes before writing any query against it
- Bind ONLY the arguments that match an index — use `_` for everything else
- If you need to query by a field combination that has no index, you may need to add one
- Row-index keys are always **arrays**, even for single-column primary keys: `["value"]` not `"value"`

```oql
;; Row index lookup — key must be an array
(Qo.Db.Prop/row-index FolderId ["https://example.com"] PageId)   ;; CORRECT
(Qo.Db.Prop/row-index FolderId "https://example.com" PageId)     ;; WRONG — never matches
```

## Execution Semantics

### No Transactions

Datastore writes happen **immediately** during execution. There is no transaction boundary and no rollback mechanism. If a term in a clause body writes data and a later term fails, the write is **not undone**.

```oql
;; If write-prop-vals succeeds but write-row-index fails,
;; the prop-vals write persists — there's no rollback
(Qo.Db.Prop/write-prop-vals PropVals _)
(Qo.Db.Prop/write-row-index FolderId RowKey PageId)   ;; if this fails, prop-vals is still written
```

### Fresh Context on Handler Invocation

When an event subscription handler fires, it creates a **fresh context** — it does NOT inherit any state from the original emitter. The handler functor must be fully self-contained, with its `:ds` reference pointing to the datastore where its clause is stored.

This means you cannot pass runtime state through subscriptions — only through the event data payload.

### println-stream Corrupts Solution Context

`println-stream` has a side effect: it modifies the solution context. If you call `println-stream` before binding your `Result` symbol, the solution context may be corrupted and `Result` becomes unresolvable.

```oql
;; BAD — println-stream before Result is bound
(println-stream Stream ["debug message"])
(= Result {"status" "done"})    ;; Result may be corrupted

;; GOOD — bind Result first, then log
(= Result {"status" "done"})
(println-stream Stream ["debug message"])
```

In practice, prefer the `print-stream` query (Omega Queries component) over raw `println-stream` — it handles this correctly.

## Performance: Thinking in Solution Trees

The key to writing efficient OQL is keeping solution sets small at each step. Every term that produces N results multiplies the solution set by N.

**Rules of thumb:**

1. **Use def-query for row processing** — each row should go through a clause boundary so intermediate symbols don't accumulate
2. **Linear lookups don't need def-query** — page tree walks, config reads produce one result each
3. **fold early** — once you have the values you need per row, fold into a list before doing more work
4. **Match indexes exactly** — extra bound variables trigger full table scans that can crash the DB
5. **Don't nest with-table-if deeply** — each level adds complexity to the solution table. Prefer flat `when` blocks or separate clauses
6. **Use `_` aggressively** — for any argument you don't need. It prevents unnecessary bindings from bloating the solution set
