[< Home](../README.md)

# How-to

Task recipes for working with Omega. Each entry is a step-by-step guide to accomplishing a specific task.

## OQL development

- [Develop OQL implementations](develop-oql-implementations.md) — REPL-driven incremental workflow for building and debugging OQL queries
- [Debug by returning early](debug-by-returning-early.md) — bisect a broken query by commenting out clauses and returning intermediate state
- [Write scratch queries](write-scratch-queries.md) — scratch queries need `(return [Result])`; without it you get a bare 500
- [Test at realistic batch sizes](test-at-realistic-batch-sizes.md) — catch batch-size-dependent bugs by testing at production limits

## OQL patterns

- [Drop rows with forced unification failure](drop-rows-with-forced-unification-failure.md) — use `(= true false)` to exclude rows from a `with-table-if` branch
- [Handle HTTP 422 in dispatch](handle-http-422-in-dispatch.md) — prevent 422 responses from leaking through status dispatch
