[< Home](../README.md) | [How-to](README.md)

# Develop OQL implementations incrementally

OQL development is REPL-driven. You build queries incrementally, one step at a time, verifying each step before adding the next. The push-run cycle is fast enough to test every single change — there is no reason to batch work.

## Prerequisites

Before writing any OQL implementation, read:

- [OQL hard rules](../reference/oql-hard-rules.md) — non-negotiable rules on forbidden datastores, forbidden operations, and the boolean-string requirement. Read this first.
- [Execution model](../explanation/oql-execution-model.md) — how OQL evaluates queries (solution sets, why terms can't be nested inside literals)
- [Built-in terms](../reference/built-ins.md) — the full built-in term reference
- [Control flow](../reference/control-flow.md) — read this before writing any `when`, `when-not`, `if`, or `with-table-if`

Skipping these leads to violating the hard rules, using non-existent terms, misusing `if` (universal, not per-row), and missing idiomatic patterns like using `(get Map "key" Val)` as a condition in `with-table-if` (the `get` itself fails on null/missing, so the else branch provides the default).

## The workflow

### 1. Start with the plumbing

Write the minimum query that proves the basic wiring works — page walk, push connector read, HTTP call. Return the raw result. Push, run, verify. Don't add anything else until this returns clean data.

```bash
qo push file.oql && qo run path -p Query -m run -a '[{}]'
```

### 2. Add one transformation at a time

Need to parse JSON? Add just `(json-stringify Parsed RawBody)` and return `Parsed`. Push, run, verify. Need to grab the first element? Add just `(get Parsed 0 First)` and return `First`. Push, run, verify. Never add two untested steps at once.

### 3. Never rewrite the file

When you get feedback on one thing, edit only that thing. Don't rewrite from scratch — you'll lose working code and reintroduce bugs you already fixed. Use incremental edits on the last known-working version.

### 4. Avoid side effects until logic is verified

Don't call `write-table` until you've returned the exact input you'd pass to it and confirmed the shape is correct. Return your would-be write-table args as the Result, inspect them, and only then swap in the actual write call. This is like dry-run mode.

```oql
;; Dry-run: inspect the write-table input before committing to the side effect
(= Result {"file-id" FolderId "data" {"header" Header "rows" Rows}})
;; After verifying the shape, swap in:
;; (Qo.Public.OqlApi.Db.Prop/write-table {"file-id" FolderId "data" {"header" Header "rows" Rows}} Result)
```

### 5. Debug by returning early

When something breaks, comment out everything after the suspected failure point and bind `Result` to the last intermediate value. This tells you exactly where the chain broke and what the data looks like at that point. See [Debug by returning early](debug-by-returning-early.md) for the full recipe.

### 6. One clause, one concern

If you need to stringify a field, get it, stringify it, set it back — three lines, return the result, verify. Don't combine stringify + write-table + count in one untested block.

## Logic tree balance

Two dimensions govern whether an OQL implementation stays manageable.

**Table header branching factor.** The number of solutions a `with-table-if` condition or clause head produces is the branching factor at that level. Keep it ≤6; even 6 requires justification. Prefer ≤4. To understand why the ceiling matters: a branching factor of 6 at three nested levels produces 6³ = 216 candidate solutions; at 4 it is 4³ = 64. Combinatorial blowup from unconstrained branching is the most common cause of logic trees that are correct in isolation but slow or undebuggable at full scale.

**Clause sub-clause count.** Each clause body should contain ≤4 sub-clauses or steps. Beyond four, the clause is doing too many things at once. Split it.

### Pre-seed map as a concrete example

The pre-seed map pattern keeps fold-body branching factor at 1. Before the fold begins, build a map that guarantees every expected key is present (using a default value for missing entries). Inside the fold body, `(get Map Key Val)` always resolves — it never widens the solution table because the key is always there. Compare this to a `with-table-if (get Map Key Val)` guard inside the fold body: if the map is missing keys, the condition fails for those rows and the else-branch fires, adding a branch. Seeding the map before the fold eliminates that branch entirely.

This also avoids the 9-entry map literal binding bug: if the map has ≥9 keys, build it in two sub-maps of ≤8 keys each and merge them before the fold. Each sub-map's `(=)` bind resolves correctly; `(merge MapA MapB Out)` combines them without going through the broken pathway.

### Cross-link

For binding rules inside `with-table-if` — which symbols must appear in the header, condition vs branch scope — see [Control flow](../reference/control-flow.md).

## What NOT to do

Don't write the entire query with HTTP call + JSON parse + field extraction + table write + error handling + count in one shot. It will fail somewhere in the middle and the 500 error won't tell you where. You'll waste time guessing and rewriting when you could have caught the issue in 30 seconds with an incremental test.

## The push-run cycle is fast

`qo push file.oql && qo run path -p Query -m run -a '[{}]'` takes seconds. There's no reason to batch work — the feedback loop is tight enough to test every single change.
