[< Home](../README.md) | [Explanation](README.md)

# Component-library testing strategy

Every `qo` component library — MAB (Manager / Researcher / Enumerate / Strategist / Copywriter / Critic / Analyst / Execute), SmartLead, GHL, HDC, the AI Connector library — currently verifies behavior by `qo push` + `qo run` against a populated, deployed datastore. That is the [push-run-verify authoring loop](../how-to/develop-oql-implementations.md) used in reverse as a test harness.

The authoring loop is the right discipline for *building* an implementation. It is the wrong primary discipline for *re-verifying* an implementation after the fact. Five symptoms make this concrete:

1. Every test cycle requires a populated DB with the right baseline state. Drift in the baseline silently changes test outcomes.
2. Agent-step tests cost real LLM tokens unless every call is gated through `test-llm-resp` (see below). One drain cycle of MAB Copywriter V1 was ~10 LLM calls; one drain of the full chain end-to-end is dozens.
3. Failures conflate logic bugs with deployment, data-shape, and network issues. A `qo run` that returns no-return could be a missing `(return [...])`, a wrong Clauses-map key, a registered-impl mismatch, a folder-structure drift, or a real body bug — and the `:failed-at nil` envelope does not distinguish them.
4. Tests cannot be re-run cheaply during refactors. A 90-second drain per regression check is enough latency to discourage refactors.
5. CI is impossible. Every test today is a manual exercise — there is no machine-runnable assertion.

This doc picks a strategy. It does not specify the harness implementation; that is downstream work.

## What "unit test" means for an OQL component library

A unit test for a `qo` component library is **a stored OQL clause that, given a synthetic input solution and a mocked or in-memory datastore context, asserts on the Result envelope or on a specific table-state delta**. It runs in isolation from the live deployment, costs no LLM tokens, and produces a pass/fail signal in seconds.

Three properties matter:

- **Isolation.** A unit test does not depend on the live `Component Installs/...` page tree, the live LLM, the live HTTP push connector, or the live production tables. Every input is named and synthetic; every output is a checkable value.
- **Determinism.** Re-running the same test twice with no code change produces the same result. No "ran fine yesterday, fails today because the production Brief page was edited."
- **Granularity.** A unit test exercises one helper clause (or one orchestrator branch), not the full OnEvent. The OnEvent end-to-end check is an integration test, not a unit test.

The push-run-verify loop in [how-to/develop-oql-implementations.md](../how-to/develop-oql-implementations.md) is a fast development feedback loop, not a unit test. The two are **complementary, not interchangeable**. Push-run-verify validates that a transformation works against the live state right now, during authoring. Unit tests validate that the implementation continues to behave correctly during refactors, dependency upgrades, and CI runs — without requiring live state.

The strategy below does **not** displace push-run-verify as the authoring loop. Authoring still pushes, runs, and verifies one transformation at a time. Unit tests are added *after* a helper is verified working, capturing the verified behavior as a re-runnable assertion.

## The existing precedent: `test-llm-resp` caller-arg

MAB Manager, Researcher, Strategist, and Copywriter already implement an ad-hoc unit-style testing pattern: every helper that dispatches an LLM call accepts a `test-llm-resp` value via the `CallerArg` map (read out of `Exec.args[0]`), and branches via `with-table-if`:

```oql
(get-in Exec ["args" 0] CallerArg)
(with-table-if (get CallerArg "test-llm-resp" TestResp)
  [TestResp LlmResp]
  (= LlmResp TestResp)
  (run-term FetchLlmResp QueriesPage PromptContent UserMessage LlmResp))
```

When the caller passes `{"test-llm-resp" {<synthetic LlmResp>}}` as the first arg, the live LLM call is bypassed and the synthetic response flows through the rest of the OnEvent body. When the key is absent, the live `FetchLlmResp` helper runs.

This pattern is **the proven precedent** for OQL unit testing. It is in production use, has been exercised across four agent steps, and lets the build phase verify post-LLM logic (parse, validate, write) without spending tokens. The strategy below generalizes it.

What `test-llm-resp` gets right:

- Substitution at the side-effect boundary, not at the consumption layer (the parse/validate/write logic is unchanged between test and live runs).
- The substitution path is in the production code, not a separate fork — there is no test/non-test build divergence.
- The caller controls the substitution via existing `Exec.args` plumbing — no new arg-passing mechanism.

What `test-llm-resp` gets wrong (or rather, what it leaves unsolved):

- It is **ad-hoc**. Each agent step re-implements the `with-table-if (get CallerArg "test-llm-resp" ...)` shape from scratch, with subtly different argument names and bind orders. There is no shared idiom or helper.
- It only addresses the LLM boundary. Datastore reads (`prop-vals-by-folder`, `read-table`), HTTP push connectors, and page-walk lookups are still live.
- It has no assertion infrastructure. The test caller has to inspect the Result envelope manually after the run; there is no machine-checkable pass/fail.
- It does not cover helpers that don't take `Exec` as an arg — i.e., most pure-logic helpers (`BatchTake5`, `ComputeStratHasMore`, `IntoSet`).

The strategy generalizes `test-llm-resp` from "ad-hoc LLM substitution" to "structured side-effect substitution with assertions."

## The strategy

### (a) What unit testing means concretely

A unit test for a component library is an OQL clause stored alongside the implementation, one of three shapes:

1. **Pure-helper test.** A clause that calls a deterministic helper (`BatchTake5`, `ComputeStratHasMore`, `ValidateCommands`, etc.) with explicit input bindings and asserts on the output. No `Exec`, no datastore, no LLM.
2. **Stateful-helper test.** A clause that calls a helper that reads or writes a datastore, given a synthetic in-memory datastore-context (see (c) below), and asserts on the output and/or the resulting table state.
3. **Branch test.** A clause that exercises one `with-table-if` branch of an OnEvent or `ProcessXBatch` end-to-end with `test-llm-resp` substitution, a synthetic Ctx, and assertions on the Result envelope.

The assertion mechanism is **a `(= ExpectedResult ActualResult)` line at the end of the test clause**. Unification failure means the test failed; the test runner observes this as a no-return on the expected-Result row. The Result envelope of the test itself is a small map: `{"test" "<test-name>" "pass" "true"}` on success, or `{"test" "<test-name>" "pass" "false" "expected" Expected "actual" Actual}` on failure (the test runner builds the failure envelope only when the unification succeeds — i.e., the inputs were close enough to bind both Expected and Actual but not equal).

### (b) Test runner design — recommendation

Three options were on the table:

- **A `qo test <impl-file>` subcommand.** New CLI verb; runs every test clause registered in the impl file.
- **A separate test library.** Tests live in a parallel page tree; some convention links impl to tests.
- **Scratch queries with a convention.** Tests are scratch `.oql` files in a known directory; a wrapper script invokes `qo run` on each.

**Recommendation: option 3 (scratch queries with a convention), promoted to option 1 (`qo test`) once the convention stabilizes.**

Rationale:

- Option 3 requires zero new infrastructure. A test is just a scratch query that ends with `(return [Result])` and follows a naming convention (e.g., `test-<helper-name>.oql` next to the impl file). The push-run-verify discipline already supports this — the only new piece is a wrapper script that runs every `test-*.oql` in a directory and checks each Result for `"pass" "true"`.
- Option 1 introduces a `qo` verb that has to know about test discovery, mock-datastore wiring, and result aggregation. That is real engineering work; deferring it until the convention has stabilized avoids designing the verb around a guess.
- Option 2 (separate test library) is the worst of both — it requires new infrastructure AND duplicates the impl page tree, creating drift risk between impl and test trees.

Concretely, the convention is:

- Tests live in the same directory as the impl, named `test-<concern>.oql`.
- Each test file is a scratch query that ends in `(return [Result])`.
- Result is a map with at minimum `{"test" "<name>" "pass" "true"|"false"}`.
- A wrapper (`qo test <dir>`, eventually) `qo push`es and `qo run`s each test, aggregates the pass/fail.

### (c) Datastore mocking

The hardest dimension. Three approaches:

- **Live DB with a test-scoped folder.** Tests write to a `Test/<test-name>/` page tree, run, and clean up. Requires a live DB but isolates test data.
- **Synthetic in-memory datastore-context.** A new harness mechanism that lets the caller pass a `{"datastore-mock" {<term-name> [<solution-rows>]}}` map in `CallerArg`; the impl's helpers read through a `read-mock-or-live` indirection.
- **Per-call mock substitution at the helper boundary.** Like `test-llm-resp` but generalized — every helper that touches a datastore takes an optional `test-<helper>-resp` caller-arg key, same `with-table-if` substitution shape.

**Recommendation: option 3 (per-call mock substitution at the helper boundary) for now. Defer option 2 (in-memory datastore-context) until the per-call shape clearly does not scale.**

Rationale:

- Option 1 (test-scoped folder) is integration testing rebranded; it does not solve the "live DB required" problem.
- Option 2 (in-memory datastore-context) requires engine support — the OQL runtime would need to recognize a mock-context argument and route `prop-vals-by-folder` / `read-table` calls through it instead of the live datastore. That is real engine work and should not block a working test convention.
- Option 3 generalizes the `test-llm-resp` precedent that already works. Each helper that reads a datastore (`ReadStratMemory`, `ReadStratFilter`, `LatestInstrContent`, `ReadReportsPlaceholder`) gains a `test-<helper>-resp` caller-arg branch with the same `with-table-if` substitution shape. The substitution boundary is the helper call site, not inside the helper. Helpers stay live; tests substitute their *outputs*.

This composes naturally: a unit test for `WriteAllStrategies` (which depends on `ReadStratFilter`) supplies `test-read-strat-filter-resp` and `test-llm-resp`, and the test's assertion is on the Result envelope of a small wrapper that invokes the function under test.

The cost is one `with-table-if` branch per substituted helper, mirrored across every test caller. This is acceptable up to ~6-8 helpers per agent step. Beyond that, option 2 (a single in-memory datastore-context) becomes worth the engine work.

### (d) Agent-step testing pattern

The MAB agent steps (Strategist, Copywriter, Critic, Analyst, Execute) all share a structural shape: `OnEvent` calls helpers that resolve Ctx, gate on filter, dispatch a batch through `ProcessXBatch`, which itself reads memory, calls the LLM, parses the response, and writes results.

The unit-test discipline for this shape is:

1. **Pure helpers** (`BatchTake5`, `ComputeXHasMore`, `IntoSet`, `ValidateCommands`) get straight pure-helper tests — input → expected output, no substitution.
2. **Datastore-reading helpers** (`ReadXMemory`, `ReadXFilter`, `LatestInstrContent`) get pure-helper tests with `test-<helper>-resp` substitution at their call sites in the impl.
3. **The orchestrator** (`OnEvent`, `ProcessXBatch`) gets one branch test per `with-table-if` branch — typically: empty-unwritten branch, non-empty drain branch, partial-drain branch, error branch (e.g., LLM truncation, parser failure).
4. **End-to-end behavior** is exercised by integration tests, not unit tests (see (f) below).

The `test-llm-resp` precedent already covers (3) for the LLM boundary. Generalizing to (2) for datastore-reading helpers is the next concrete step.

### (e) Coverage expectations

**Recommendation: every helper named in the OnEvent / `ProcessXBatch` capture list gets at least one unit test. Pure helpers get a happy-path test plus one edge-case test (empty input, single-element input, etc.). Orchestrators get one test per `with-table-if` branch.**

Not every line needs coverage — the OQL engine is the test for the language layer. What needs coverage is **the application logic**: the validation rules, the parse-response logic, the count/length idioms (e.g., the `(length AllKeys Total)` dedupe gotcha — see [agent-step-write-shapes.md](agent-step-write-shapes.md)), the early-return guards, the cold-start gates.

Refactor pressure is the test of coverage adequacy: if you can refactor a helper from inline to extracted (or vice versa) without a test failing, the test was at the wrong granularity. Conversely, if every refactor breaks twelve tests, the tests are coupled to implementation rather than to behavior.

### (f) Integration vs unit — when to run which

- **Unit tests run on every push, in every CI invocation, in seconds.** Their job is to catch logic regressions during refactors and dependency upgrades.
- **Integration tests run before a deploy and after a schema change, in minutes.** Their job is to validate that the impl + datastore + push-connector wiring produces the expected end-to-end behavior. This is the existing push-run-verify drain (e.g., MAB Copywriter V1's 27-treatment drain) — promoted from "what authoring does" to "the full-deploy gate."
- **The push-run-verify authoring loop is neither.** It is a development feedback mechanism. Once a helper is verified working, the verified behavior is captured as a unit test; the authoring loop moves to the next helper.

The three loops compose: **author with push-run-verify → land a unit test capturing the behavior → run unit tests on every change → run integration tests before deploy.** The strategy is not "replace push-run-verify with unit tests"; it is "complement push-run-verify with re-runnable assertions so refactors are cheap."

## What this strategy is not

- **Not a mandate to retro-test every existing component library before adding new features.** Add unit tests for new helpers as they are written; back-fill old helpers as refactor pressure surfaces them.
- **Not a replacement for [develop-oql-implementations.md](../how-to/develop-oql-implementations.md).** That doc is the canonical authoring discipline. It still applies. Unit tests are layered on top, not in place of.
- **Not a gate on shipping.** A V1 implementation with no unit tests is acceptable if the push-run-verify drain succeeded. The unit tests come during the next refactor pass — the moment when the cost of re-running the integration drain exceeds the cost of writing the unit tests.
- **Not a specification for the harness.** The shape of `qo test`, the wrapper script, the assertion-failure envelope format — those are downstream design decisions. This doc fixes the strategy; the harness implementation interprets it.

## Open questions for the harness implementation

These are deliberately left for the harness design phase, not decided here:

- How does the test runner discover tests? Convention-based filename match (`test-*.oql`)? An explicit registry?
- How does an assertion-failure envelope flow back from `qo run` to the wrapper? `(= Expected Actual)` failing produces no-return; the wrapper has to interpret that as "test failed" rather than "test errored."
- Do datastore-mock substitutions need a way to assert on the *call shape* (i.e., "the helper called `prop-vals-by-folder` with these args"), or only on the resulting Result envelope?
- Is there a standard mock-fixture format (a `test-fixtures/<agent>.edn` file with synthetic LLM responses, synthetic Memory rows, etc.) so multiple tests can share inputs?

These are real questions; deferring them does not weaken the strategy. The strategy's load-bearing claim is "generalize `test-llm-resp` to all side-effect boundaries, store tests as scratch queries with a convention, layer on top of push-run-verify rather than replacing it." The harness design works out the rest.

## Related

- [how-to/develop-oql-implementations.md](../how-to/develop-oql-implementations.md) — the authoring loop. Unit tests complement this; they do not replace it.
- [how-to/debug-by-returning-early.md](../how-to/debug-by-returning-early.md) — the Result-as-print-statement idiom. Unit tests are the same idiom promoted to a re-runnable assertion.
- [how-to/write-scratch-queries.md](../how-to/write-scratch-queries.md) — scratch-query shape (every scratch query needs `(return [Result])`). Unit tests under the recommended convention are scratch queries.
- [explanation/agent-step-plan-shape.md](agent-step-plan-shape.md) — the flat-helpers shape that makes per-helper unit testing tractable. Helpers that are 5-10 lines flat are testable; 90-line nested-with-table-if blocks are not.
