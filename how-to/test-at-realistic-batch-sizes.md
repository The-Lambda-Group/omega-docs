[< Home](../README.md) | [How-to](README.md)

# Test at realistic batch sizes

How to catch batch-size-dependent bugs by testing OQL queries at production-scale limits, not `limit=1`.

## The rule

**Always test at realistic batch sizes before declaring working.** If your pipeline runs at 100 rows per batch, test at 100 — not at 1.

## Why this matters

Several OQL constructs have behavior that depends on row count. A query that works at `limit=1` can fail at `limit=100`:

- **`with-table-if` schema collapse** (see [Control flow — Preserving per-row identity](../reference/control-flow.md#preserving-per-row-identity)) only bites when the collapse's merge behavior matters — and "matters" depends on how many rows share schema values. At `limit=1` the collapse is invisible.
- **`fold` over unique values** (see [Reducers — fold pitfalls](../reference/reducers.md#fold-pitfalls)) at `limit=1` has one row with one value, and `Sum = 1` looks correct by coincidence.
- **Mixed-literal symbols** may fail only when the literal actually differs across rows.
- **422 / non-2xx responses** (see [Handle HTTP 422 in dispatch](handle-http-422-in-dispatch.md)) only surface when a batch contains a row that triggers them.

## Diagnostic sequence when a batch-size bug appears

1. Reproduce at `limit=1`. Does it work? (Usually yes.)
2. Reproduce at `limit=100`. Does it break? Binary-search down to the smallest batch size that reproduces.
3. Identify the row that triggers the break at that batch size. Is it a specific row's data, or is it an interaction effect across rows?
4. If it's a specific row: the bug is probably data-shape (a 422, a bad value, an empty field).
5. If it's an interaction effect: the bug is probably one of the schema-collapse / fold / mixed-literal footguns listed above.
