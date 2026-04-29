[< Home](../README.md)

# How-to

> **Confidentiality: PUBLIC.** All content in this directory is public platform documentation.

Task recipes for working with Omega. Each entry is a step-by-step guide to accomplishing a specific task. These are "how do I ..." docs -- if you need "what is ..." lookup, see [reference/](../reference/README.md); if you need "why does ..." understanding, see [explanation/](../explanation/README.md).

## Information ownership

This bucket owns the **public OQL authoring methodology** (the push-run loop), **OQL failure triage methodology**, and **language-level coding patterns** for the public OQL surface (early-return debugging, scratch query structure, batch-size discipline, row-drop idioms, HTTP dispatch handling). It does not own deployment, CLI usage, MCP server setup, component-implementation authoring beyond the language layer, or any project-specific operational procedure — those live in their respective project repos. Private operational procedures live in `omega-knowledge-base`.

## Navigation map

### Methodology (read first)

This doc is the cross-cutting methodology for all OQL work -- authoring and triage in both directions. It is not a peer of task-specific how-tos; it describes the loop every OQL session runs. Read it before descending further.

- [Develop OQL](develop-oql.md) -- The OQL operating manual. Covers the probe-don't-build mindset (the "verb shift" for LLMs), the `Result`-as-inspection-channel mechanic (no print statements in OQL), the authoring loop (forward: push → run → verify one transformation at a time, dry-run side effects), the triage loop (backward: six artifact-producing rules -- min repro, stack-frame-as-address-not-diagnosis, prior-bug-as-hypothesis-not-conclusion, layer ruled-in/ruled-out table, "I don't know yet" as preferred state, discipline-decay-under-momentum), the wall-clock-as-cardinality-signal heuristic with reference points, and two worked examples (Enumerate authoring + 2026-04-24 add-protocol triage). Read before writing **or** debugging any OQL. Contains an "Operating manual (re-read every session)" 8-rule TLDR at the top -- if you only have time for one thing, read that section. Keywords: push-run, incremental, REPL, workflow, dry-run, write-table verification, debug, triage, Mango, full-scan, failure, minimum reproduction, min repro, narrow, bisect, diagnosis, WRITING_SYMBOLS, postmortem, LLM failure modes, build-mode, probe-mode, cardinality, wall-clock.

---

### OQL development

- [Debug by returning early](debug-by-returning-early.md) -- How to bisect a broken query by commenting out clauses and binding `Result` to a tuple of intermediate variables. Covers why tuples are better than `throw` (multi-row visibility), the HTTP debugging sub-recipe (inspecting request maps alongside responses to catch symbol-leak bugs), and iteration discipline (one change per run, include per-row identity). Keywords: debug, bisect, early return, tuple, throw vs tuple, HTTP debugging, intermediate state, comment out.

- [Write scratch queries](write-scratch-queries.md) -- Scratch queries (bare term sequences run with `qo run` or `qo query`) require an explicit `(return [Result])` as the last line. Without it, the API returns a bare HTTP 500 with no diagnostics. This doc explains why the 500 is hard to debug and what to check first. Keywords: scratch query, return, bare 500, no error message, structurally incomplete.

- [Triage `omega/query/no-return`](triage-no-return.md) -- No-return has three modes: scratch-file (missing `(return [Result])`), dispatch-layer (four causes: caller `-a` arity does not match protocol spec arity, wrong Clauses-map keys, malformed protocol spec, impl not stored), and body-level zero-solutions (the body ran but a term inside it produced zero solutions through unification — undefined clause call, pkey lookup miss, output variable never bound, `fold` over an empty input). Dispatch-layer and body-level zero-solutions modes share an identical envelope (`:failed-at nil`, no JVM trace, the misleading "Scratch queries require..." message); walk the four dispatch-layer checks first because they are cheaper, then bisect the body if all pass. The `:failed-at nil` signal does **not** distinguish dispatch-layer from body-level zero-solutions -- it only rules out the exception-throwing body case (which has frames). Read this when no-return surfaces from a `qo run` against a stored impl (or any `Qo.Public.Api.Run/*` caller) -- the "add a return statement" advice does not apply because the impl is a clause. Keywords: no-return, dispatch, arity, Clauses map, protocol spec, failed-at nil, silent zero solutions, qo run, body bisection, undefined clause, pkey miss, fold empty input.

- [Test at realistic batch sizes](test-at-realistic-batch-sizes.md) -- Why `limit=1` testing masks bugs that only appear at production batch sizes. Covers which OQL constructs are batch-size-dependent (`with-table-if` schema collapse, `fold` over unique values, mixed-literal symbols, non-2xx HTTP responses), and a diagnostic sequence for isolating batch-size bugs. Keywords: batch size, limit, production testing, schema collapse, fold pitfall, 422.

### OQL patterns

- [Drop rows with forced unification failure](drop-rows-with-forced-unification-failure.md) -- The `(= true false)` idiom for deliberately excluding rows from a `with-table-if` branch. Covers when to use it (filtering HTTP errors, sentinel values, debug short-circuits), how it works (unification failure removes the row from the solution), and anti-patterns (don't use in else-branches, don't use at top level, don't confuse with `throw`). Keywords: drop rows, filter, unification failure, `(= true false)`, with-table-if, exclude, silent drop.

- [Handle HTTP 422 in dispatch](handle-http-422-in-dispatch.md) -- How to prevent 422 and other unexpected HTTP status codes from leaking through `with-table-if` dispatch branches and causing `WRITING_SYMBOLS` errors. Covers the row-drop filter pattern, the general rule for HTTP status dispatch (enumerate all statuses or use a catch-all filter), and how to diagnose `WRITING_SYMBOLS` errors by tracing unbound symbols back to dispatch gaps. Keywords: HTTP 422, WRITING_SYMBOLS, status dispatch, unbound symbols, dispatch gap, catch-all filter.
