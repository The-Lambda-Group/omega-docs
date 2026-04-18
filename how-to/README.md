[< Home](../README.md)

# How-to

> **Confidentiality: PUBLIC.** All content in this directory is public platform documentation.

Task recipes for working with Omega. Each entry is a step-by-step guide to accomplishing a specific task. These are "how do I ..." docs -- if you need "what is ..." lookup, see [reference/](../reference/README.md); if you need "why does ..." understanding, see [explanation/](../explanation/README.md).

## Information ownership

This bucket owns public OQL development workflows and coding patterns. Project-specific how-to docs (e.g., CLI usage, MCP server setup) live in their respective project repos. Private operational procedures live in `omega-knowledge-base`.

## Navigation map

### OQL development

- [Develop OQL implementations](develop-oql-implementations.md) -- The core development workflow. Covers the incremental REPL-driven push-run cycle (`qo push && qo run`), the rule of adding one transformation at a time, never rewriting a working file, avoiding side effects until logic is verified, and the one-clause-one-concern discipline. Start here if you are writing OQL for the first time. Keywords: push-run, incremental, REPL, qo push, qo run, workflow, dry-run, write-table verification.

- [Debug by returning early](debug-by-returning-early.md) -- How to bisect a broken query by commenting out clauses and binding `Result` to a tuple of intermediate variables. Covers why tuples are better than `throw` (multi-row visibility), the HTTP debugging sub-recipe (inspecting request maps alongside responses to catch symbol-leak bugs), and iteration discipline (one change per run, include per-row identity). Keywords: debug, bisect, early return, tuple, throw vs tuple, HTTP debugging, intermediate state, comment out.

- [Write scratch queries](write-scratch-queries.md) -- Scratch queries (bare term sequences run with `qo run` or `qo query`) require an explicit `(return [Result])` as the last line. Without it, the API returns a bare HTTP 500 with no diagnostics. This doc explains why the 500 is hard to debug and what to check first. Keywords: scratch query, return, bare 500, no error message, structurally incomplete.

- [Test at realistic batch sizes](test-at-realistic-batch-sizes.md) -- Why `limit=1` testing masks bugs that only appear at production batch sizes. Covers which OQL constructs are batch-size-dependent (`with-table-if` schema collapse, `fold` over unique values, mixed-literal symbols, non-2xx HTTP responses), and a diagnostic sequence for isolating batch-size bugs. Keywords: batch size, limit, production testing, schema collapse, fold pitfall, 422.

### OQL patterns

- [Drop rows with forced unification failure](drop-rows-with-forced-unification-failure.md) -- The `(= true false)` idiom for deliberately excluding rows from a `with-table-if` branch. Covers when to use it (filtering HTTP errors, sentinel values, debug short-circuits), how it works (unification failure removes the row from the solution), and anti-patterns (don't use in else-branches, don't use at top level, don't confuse with `throw`). Keywords: drop rows, filter, unification failure, `(= true false)`, with-table-if, exclude, silent drop.

- [Handle HTTP 422 in dispatch](handle-http-422-in-dispatch.md) -- How to prevent 422 and other unexpected HTTP status codes from leaking through `with-table-if` dispatch branches and causing `WRITING_SYMBOLS` errors. Covers the row-drop filter pattern, the general rule for HTTP status dispatch (enumerate all statuses or use a catch-all filter), and how to diagnose `WRITING_SYMBOLS` errors by tracing unbound symbols back to dispatch gaps. Keywords: HTTP 422, WRITING_SYMBOLS, status dispatch, unbound symbols, dispatch gap, catch-all filter.
