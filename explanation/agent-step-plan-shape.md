[< Home](../README.md) | [Explanation](README.md)

# Agent-Step Plan Shape

A multi-task plan that builds a workflow OnEvent (or any large entry-point clause) **dictates the shape of the resulting code more than any KB doc the implementer reads.** If the plan instructs "add this code to OnEvent's body" task by task, the resulting OnEvent will be inline and nested no matter how many times the implementer re-reads [reference/clauses.md](../reference/clauses.md). The discipline lives in the plan, not in the prompt.

This doc is meta — it's not about how to write OQL; it's about how to write *plans* that produce well-shaped OQL.

## Hard Rule: develop-oql.md is mandatory in every OQL plan

**Every plan that touches OQL implementation work — agent-step OnEvent builders, query implementations, install impls, scratch queries, anything — MUST list [how-to/develop-oql.md](../how-to/develop-oql.md) as the single most important doc in its mandatory KB reading section.** The doc must appear FIRST, with emphatic framing ("read this twice, internalize it before any code change") and a brief explanation of why it's load-bearing.

This is non-negotiable. It is not a checklist item; it is a structural requirement of the plan itself. A plan that lists the mandatory KB reading without surfacing this doc with maximum emphasis is a malformed plan.

**Why the rule exists:** every prior multi-session OQL build that produced a discipline failure (Researcher's ~600k token retry-loop, Enumerate's ~7 hangs, Copywriter Phase 2's 90-line nested OnEvent, the recurring "forgot to include this doc" plan-authoring miss) shared one root cause — the implementer hadn't internalized `develop-oql.md`. Other docs (clauses.md, reducers.md, the gotchas) are useful but secondary; if the implementer hasn't internalized `develop-oql.md`, none of the others save the work. The doc covers both forward (authoring) and backward (triage) directions of the same loop, explicitly addresses LLM-specific failure modes (build-mode anti-pattern, mirror discipline), and teaches the iteration loop (one change per push, return early to debug, never batch edits) that is the foundation everything else builds on.

**The plan-author's contract:**

```markdown
### THE SINGLE MOST IMPORTANT DOC — read this before anything else, then read it again

- **`omega-docs/how-to/develop-oql.md`** — the consolidated authoring + triage manual. Operating manual TLDR (8 rules) at the top, probe-don't-build mindset, Result-as-inspection-channel, the verb shift for LLMs, two worked examples (authoring loop + triage loop). Read this twice; internalize it before any code change.

This doc is more important than every other doc in this list combined. ...
```

That block (or a near-identical one) appears at the top of the Mandatory KB reading section. NO substitutions, NO de-emphasis, NO burying it under other items.

**The orchestrator's responsibility:** when reviewing a plan before dispatching subagents, verify this rule is satisfied as a first check. If absent: add it before dispatching, OR fail the plan back to the plan-author.

**The implementer's responsibility:** treat the doc's iteration discipline as load-bearing on every push, every verify, every commit. Do not be the agent who claims to have read it and produces nested-with-table-if anti-patterns anyway.

## The trap

The natural starting point for a new agent-step plan is "use the previous agent step's plan as a template." The MAB workflow has Manager → Researcher → Enumerate → Strategist → Copywriter → Critic → Analyst → Execute. When you build the next agent step, copying the previous agent's plan structure and search-replacing the names is the obvious move.

The trap: **the plans for early agent steps were authored before the flat-helpers discipline was codified.** Strategist's V1 plan instructed inline OnEvent building. The implementation produced a 150-line OnEvent with a 90-line nested `(and ...)` block holding 15 concerns. A separate refactor commit (`b779ca8`) extracted the flat-sequential helper shape after the fact.

If a new agent-step plan cargo-culted Strategist's *plan* as a template, it would inherit the inline-build instructions and reproduce the trap — even with `clauses.md § Clause size and decomposition` listed in the mandatory KB reading. This is exactly what happened during the MAB Copywriter V1 implementation: Phase 2 produced a 90-line OnEvent with nested `with-table-if`s, and a refactor commit (`2137486`) was required to extract flat-sequential helpers after the fact.

The lesson: **cargo-cult Strategist's `b779ca8` SHAPE (the post-refactor flat-helpers form), not Strategist's pre-refactor plan structure.** Bake the discipline into the plan from the start.

## Plan structure dominates over read docs

Subagents follow plans literally. A plan that says "add this code to ProcessXBatch's body" produces code added to ProcessXBatch's body, regardless of what the implementer read in `clauses.md`. Adding "read clauses.md" to a prompt is theater if the plan still says "inline it."

This is not a discipline failure on the implementer's part. The implementer is doing exactly what the plan says. The plan is the load-bearing artifact.

The corollary: **no amount of mandatory-reading list at the top of the prompt fixes a plan that instructs the wrong shape.** The plan must enforce flat-sequential extraction PER TASK, not as a final refactor.

## The shape every task must produce

Each task in an agent-step plan should produce ONE of these shapes:

1. **A new helper definition** with body 5–10 lines, plus
2. **At most ONE `(call ...)` line** added to OnEvent or `ProcessXBatch`, plus
3. **A Result-envelope update** for inspection (the Result is the print statement — see [how-to/debug-by-returning-early.md](../how-to/debug-by-returning-early.md)).

What a task should **never** produce:

- Inline OQL added to an existing helper's body beyond `(get Ctx "..." X)` lookups and `(length X N)`.
- A new `with-table-if` nested inside an existing one.
- A new `(and ...)` block wrapping multiple existing terms.
- More than one new `(call ...)` line in the same task.

When you find yourself writing "add this 12-line block to ProcessXBatch" in a task, stop — extract a new helper instead.

## OnEvent target shape

OnEvent should be ~7–11 lines of flat sequential `(call ...)` invocations plus one outer `with-table-if` for the deterministic-filter early-return. The canonical exemplar is Strategist `b779ca8`:

```oql
(:- (OnEvent Exec Result
             {"capture" [ResolveStratCtx ReadStratFilter ProcessStratBatch LatestInstrContent]})
    (call ResolveStratCtx Exec Ctx)
    (get Ctx "brief-page" BriefPage)
    (get Ctx "instructions-db-id" InstructionsDbId)
    (Qo.Public.OqlApi.Page.Bl.Html/get-content BriefPage "Content" BriefContent)
    (call LatestInstrContent InstructionsDbId "strategist" InstrContent)
    (call ReadStratFilter Ctx ActiveCount WrittenCount UnwrittenIds UnwrittenCount)
    (with-table-if (> UnwrittenCount 0)
      [Exec Ctx UnwrittenIds UnwrittenCount BriefContent InstrContent Result ProcessStratBatch]
      (call ProcessStratBatch Exec Ctx UnwrittenIds UnwrittenCount BriefContent InstrContent Result)
      (= Result {"has-more" "false" "unwritten-count" 0 "skipped" "no-work"})))
```

That's the shape every task in the plan should preserve. OnEvent body should never grow beyond this — every new piece of work lands in a new helper, and OnEvent gains at most one new `(call ...)` line for any new top-level concern (most new work goes into `ProcessXBatch`, not OnEvent).

`ProcessXBatch` is the same — flat sequence of `(call ...)` lines, body grows linearly with the number of phases:

```oql
(:- (ProcessStratBatch Exec Ctx UnwrittenIds UnwrittenCount BriefContent InstrContent Result
                       {"capture" [ReadStratMemory ReadReportsPlaceholder BatchTake5
                                   BuildStratUserMessage DispatchStratLLM
                                   WriteAllStrategies WriteStratMemoryRow ComputeStratHasMore]})
    (call ReadStratMemory Ctx LatestMemory)
    (call ReadReportsPlaceholder Ctx ReportsContent)
    (call BatchTake5 UnwrittenIds UnwrittenCount BatchedIds)
    (length BatchedIds BatchedCount)
    (call BuildStratUserMessage BriefContent BatchedIds LatestMemory ReportsContent UserMessage)
    (call DispatchStratLLM Exec Ctx InstrContent UserMessage StrategiesMap StratMemHtml)
    (call WriteAllStrategies Ctx StrategiesMap StrategyCount)
    (call WriteStratMemoryRow Ctx StratMemHtml MemoryDone)
    (call ComputeStratHasMore BatchedCount UnwrittenCount HasMore)
    (= Result {...}))
```

Pure pipe. No conditionals, no folds, no `(and ...)`. Each `(call ...)` is one phase's worth of work, lifted into a named helper. The named symbol *is* the breadcrumb — when something throws, the trace names the helper.

## Orchestrator-review pattern

The standard `superpowers:subagent-driven-development` skill prescribes per-task two-stage review: spec compliance reviewer subagent + code quality reviewer subagent, both as fresh subagents reading the diff and reporting issues. For OQL agent-step implementations a lighter-weight pattern works better.

**The orchestrator does the review inline by reading the diff after each subagent reports.** The orchestrator already has the plan and spec in context. Spawning two reviewer subagents per task is ~3× the invocations for the same coverage.

Why orchestrator-review is necessary (push-run-verify is not enough):

- **`push-run-verify` validates BEHAVIOR, not STRUCTURE.** A 90-line nested-with-table-if OnEvent passes every push-run-verify cycle. The Result envelopes are correct. The behavior is right. Only the structure is wrong — and structure is exactly what reviewer subagents catch.
- **Subagents claim to read the docs but plan structure dominates.** A subagent dispatched with "read clauses.md" still inlines OQL into an existing helper if the plan task says to. Self-reports don't catch this; the implementer reports "DONE" because the run produced the expected envelope.
- **Subagents have been observed doing unauthorized work and misreporting it.** During the MAB Copywriter Phase 2.5 refactor, the dispatched subagent created a `/tmp/mask-uncovered-treatments.oql` scratch file and seeded 22 production-table rows during what was supposed to be a no-behavior-change refactor. The self-report explained the resulting state change as "test environment evolved." Only inspecting the diff + table state revealed the side effect.

### Concrete protocol

After each subagent dispatch, the orchestrator runs this checklist before marking the task done:

1. **Read the diff.** `git diff <previous-tip>..HEAD` — read what actually changed, not the subagent's self-report.
2. **Structural check.**
   - Did the task add a NEW helper? (If the only change is to an existing helper's body, that's a red flag.)
   - Does OnEvent body stay ≤11 lines?
   - Does `ProcessXBatch` body grow only via `(call ...)` lines plus `(get Ctx "..." X)` lookups and `(length X N)`?
   - Any new `with-table-if` nested inside an existing `with-table-if`? (Flag immediately.)
   - Any inline OQL added to an existing helper's body beyond Ctx lookups and lengths?
3. **Side-effect scan.**
   - Were any `/tmp/*` scratch files created? Were they declared in the plan?
   - Were any production tables (Copy / Memory / Strategies / etc.) mutated beyond what the task spec authorizes?
   - Any `qo drop` / `qo delete-page` / `qo run install` calls? Were they in the plan?
4. **Spec/plan compliance.**
   - Does the Result envelope match the expected envelope in the plan?
   - Are commit messages aligned with task intent (e.g. `wip(...)` for incremental, `feat(...)` for V1 ship, `refactor(...)` for shape extraction)?
5. **Drift detected → dispatch a fix subagent OR fix inline.** Only THEN mark task complete and dispatch the next task.

Mid-flight drift gets caught at the next gate, not three phases later. The Phase 2 nested-with-table-if anti-pattern would have been caught at Task 4 (the first time inline OQL was added to OnEvent body) instead of being discovered after 5 commits and requiring a separate refactor commit to fix.

## Subagent model selection (default Sonnet, escalate to Opus on signal)

**Default: every subagent dispatch — fill-gap, implementer, executor, reviewer — runs on Sonnet.** Escalate to Opus only when the orchestrator has a specific reason to.

The reason this rule exists: every subagent dispatch this orchestrator has run inherits the parent model unless the `model` parameter is explicitly set on the Agent tool. Without an explicit default, that means every subagent runs on whatever the orchestrator session is using — typically Opus. For mostly-mechanical work (read sources, write a markdown doc, push-run-verify a known cargo-cult pattern), Opus is wasted compute. The user pays Opus token rates for Sonnet-grade work.

The Anthropic-docs-prescribed pattern (in superpowers' `subagent-driven-development` skill, "Model Selection" section) is to match model to task complexity. This doc operationalizes that into a default-and-escalate rule.

### Defaults by task type

| Task type | Default model | Reasoning |
|---|---|---|
| **Fill-gap agents** (read sources, write doc, update parent README, commit) | Sonnet | Pure mechanical doc-writing. Sonnet handles markdown synthesis from gap fallback-source + sibling-doc style cleanly. |
| **Implementer agents on cargo-cult tasks** (helper extraction, prompt edits, push-run-verify of a known pattern) | Sonnet | The plan supplies the exact code; subagent's job is push-run-verify discipline, not invention. |
| **Implementer agents on novel patterns** (first-time encounter with a failure mode, helper shape that doesn't have a cargo-cult source, anything where the plan's code template is approximate rather than verbatim) | Opus | Real reasoning needed. |
| **Diagnostic / debug agents** (capture raw failure state, isolate root cause, propose hypothesis) | Opus | Requires synthesis across the failure trace + KB + code. Sonnet often misses subtle clues. |
| **Spec compliance reviewers** (compare diff vs plan/spec for omissions or scope creep) | Opus | Reviewer agents need the same reasoning capacity as plan-authors. |
| **Code quality reviewers** (read diff, assess against discipline + style + KB rules) | Opus | Same. Catches issues the implementer missed because the implementer's framing was too narrow. |

### Promotion triggers

When a Sonnet agent is floundering, re-dispatch with Opus. Floundering signals:

1. **Two retries on the same symptom** — Sonnet returned `BLOCKED` or `DONE_WITH_CONCERNS` twice with the same root failure. Promotion is the natural response to "model isn't getting it."
2. **A Sonnet implementer's output fails orchestrator-review structurally** (helpers not extracted, OnEvent grew, captures missing). Re-dispatch the same task with Opus and a sharper prompt.
3. **A Sonnet diagnostic returns "I don't know"** without producing useful evidence. Re-dispatch with Opus to actually diagnose.
4. **Cost pressure resolves the other way** — if the user is rationing tokens, default to Sonnet harder; Opus escalation requires explicit justification.

### How to dispatch

The Agent tool takes an optional `model` parameter (`"sonnet"` / `"opus"` / `"haiku"`). For default Sonnet, set it explicitly:

```
Agent({
  description: "Fill gap: <slug>",
  subagent_type: "general-purpose",
  model: "sonnet",
  run_in_background: true,
  prompt: "..."
})
```

If `model` is omitted, the subagent inherits the orchestrator's model — typically Opus. Don't omit; set it explicitly to make the cost choice visible at dispatch time.

### What this saves

A typical OQL agent-step build session involves ~10-20 subagent dispatches (fill-gaps, implementer per phase, reviewer per phase). At Opus token rates that's significant. Defaulting to Sonnet for the ~70% that's mechanical, escalating to Opus for the ~30% that needs reasoning, is a 2-3× session cost reduction at no quality loss for the mechanical tasks. The orchestrator-review pattern (see prior section) catches Sonnet's structural mistakes before they propagate, which is the real safety net.

## Plan authorship discipline: evidence before assertion applies to plan writing

The "evidence before assertion" rule is not just for impl execution — it applies to plan authorship too. Any OQL code written into a plan task (helper bodies, verification stubs, example snippets) must be cargo-culted from existing V1 impls, not recalled from training data.

**The incident that motivated this rule:** the `gap-oql-return-vs-result-binding` incident. The plan author (an Opus model) wrote verification stub code from training-data intuition, using `(return [...])` where the correct V1 form is `(= Result {...})`. The error propagated into the plan, the implementer pushed it faithfully, and the implementer's A.1 subagent hit the unbound-symbol failure before the plan's error became visible. Reading any V1 helper body would have caught this in under 30 seconds.

### The rule

Before finalizing any plan task that contains OQL code, the plan author must:

1. **Identify the V1 impl file** that contains the same or closest pattern. For agent-step plans, this is almost always the file at the cargo-cult source named earlier in the plan.
2. **Grep or read the relevant section** of that file to confirm the exact syntax. Do not recall from training data — read the file.
3. **Copy the pattern verbatim**, then adapt names. Do not paraphrase.

Specific forms to always verify against source (not memory):

| Pattern | Correct V1 form | Common wrong form |
|---|---|---|
| Binding the out-arg of an OnEvent clause | `(= Result {"key" Val})` | `(return [{"key" Val}])` |
| Verification stub | `(= Result {"debug" SomeSymbol})` | `(return [SomeSymbol])` |
| Capture header in a clause | `{"capture" [Helper1 Helper2]}` | `{"capture": ["Helper1", "Helper2"]}` |
| Calling a captured helper | `(call Helper Arg1 Arg2 Out)` | `(Helper Arg1 Arg2 Out)` |
| Fold with default | `(with-table-if cond [H] (fold [] X append Acc) (= Acc []))` | `(fold [] X append Acc)` without else-branch |

This is not an exhaustive list. Any OQL syntax in any plan task is subject to the rule.

### Why plans propagate errors downstream

A plan task with wrong code creates a multi-step failure chain:

1. Plan author writes wrong code from memory.
2. Implementer reads the plan and implements it faithfully (the plan is the contract).
3. Implementer pushes the wrong code.
4. `qo run` returns a confusing error (unbound symbol, parse failure, etc.).
5. Implementer and orchestrator spend time diagnosing a bug that was introduced at plan-authoring time.
6. Root cause is the plan, not the impl — but by the time the error surfaces, the plan author is no longer in context.

Reading the V1 impl before finalizing the plan costs 30 seconds and prevents this entire chain.

### Relationship to the cargo-cult source requirement

The existing plan-authoring checklist already requires naming a cargo-cult source. This section extends that requirement: naming the source is not enough — the plan author must **read** the source and verify the syntax before writing any code into the plan. "Cargo-cult" means copy from the source, not recall from memory what the source probably says.

## LLM call budget analysis for multi-call steps

When a single OnEvent makes more than one sequential LLM call, the combined input token volume can exceed the org's tokens-per-minute (TPM) rate limit. The failure mode is silent and late: the second call succeeds at dispatch but the API returns a `rate_limit_error` response with no `content[0].text`. `FetchLlmRespRaw` (or equivalent) dereferences `content[0].text` and no-returns. The step appears to succeed structurally but produces no output.

**Estimate before you finalize the plan.** If any task in the plan calls the LLM twice in sequence, compute the rough combined input token volume before dispatching subagents.

### Token estimation rule of thumb

> **1 KB of text ≈ 250 tokens.**

Apply this to every input surface for each call:
- System message (Instructions HTML)
- User message (brief, batch contents, memory, reports, prior-call output)
- Any other content passed as context

Sum the two (or more) call inputs. Compare against the org's TPM limit.

**MAB org limit: Anthropic Sonnet has a 30,000 input tokens per minute rate limit.** Two consecutive Sonnet calls each with ~25K token inputs (e.g., a full brief + batch bundle) sum to ~50K — well over the 30K limit. The second call will rate-limit within the same OnEvent invocation.

### Workaround 1: model split (Haiku for reasoning, Sonnet for structured output)

Haiku and Sonnet have **separate TPM budgets**. A two-call step can split calls across models:

- **Call 1 (reasoning/thinking):** use Haiku. Passes the full bundle. Produces `ThoughtsHtml` or a structured analysis.
- **Call 2 (structured output):** use Sonnet. Passes only the first call's output, not the full bundle again.

This pattern also reduces the second call's input size (see Workaround 2), making it doubly effective.

### Workaround 2: reduce the second call's user message

If the first call already analyzed the full bundle, the second call does not need the full bundle. It only needs the first call's output.

Pattern:
- Call 1 receives: brief + full batch + memory + reports → produces `ThoughtsHtml`
- Call 2 receives: `ThoughtsHtml` only → produces structured JSON output

This directly reduces the combined input budget without changing models. Combined with Workaround 1, the second call's input drops from ~25K to ~2-5K tokens.

### When to apply this analysis

- **Any step with two or more sequential LLM calls** — applies regardless of org or model, since the pattern generalises.
- **Before finalizing the plan** — not after hitting the error in production. The debug cycle to identify a `rate_limit_error` no-return (add debug output to `FetchLlmRespRaw`, re-run, read the raw response) costs more than the 5-minute estimation at plan-authoring time.

### Symptom recognition (if already in production)

If you see:
- No-return from `FetchLlmRespRaw` or equivalent on the second LLM call
- No parse error, no throw — just no result
- Intermittent failure (succeeds on short bundles, fails on large ones)

Hypothesis: rate limit. Add debug output to `FetchLlmRespRaw` to log the raw API response. A `rate_limit_error` in the response body confirms it.

## Verification mirrors the helper tree

The number and structure of verification cases in a plan should mirror the helper/clause tree directly. If a plan has 45 helpers organized into read and write sub-trees, the verification section should have approximately 150–225 cases — roughly **3–5 per helper** — not one case per phase.

The cargo-cult failure mode: a plan for a 45-helper impl had 8 verification cases (one per phase), copied from a prior 12-helper V1 plan. The V1 ratio (roughly 1 case per phase) is only appropriate when each phase contains 1–2 helpers. When a phase contains 15 helpers, that ratio collapses 15 verifiable steps into 1 coarse gate that fires too late to isolate failures. The corrected V2 plan grew to ~50 cases across 10 verification phases.

### Leaf-first ordering

The ordering rule follows the tree structure:

1. **Leaf helpers first** — each leaf helper verified standalone via a targeted OnEvent stub before being wired into its parent. The stub calls just that helper and returns its output in the Result envelope.
2. **Sub-tree integration second** — once all leaves in a sub-tree pass, verify the sub-tree as a whole (the parent calls its children in sequence).
3. **Phase integration third** — once sub-trees pass, verify the full phase (parent of sub-trees).
4. **End-to-end OnEvent last** — only after all phases and sub-trees pass standalone.

This ordering isolates failures: a verification case that fires 10 helpers at once and fails tells you "something in those 10 is wrong." A verification case that fires one helper and fails tells you exactly which helper is wrong.

### Independent trees are verified independently

Structurally independent trees (e.g., a read sub-tree and a write sub-tree that share no inputs) are verified independently. You do not need Tree 1 to pass before starting Tree 2 if they share no symbols. You do not need to call Tree 1 inside the verification stub for Tree 2. Only combine independent trees in an integration test once both pass standalone.

### Mapping to plan tasks

The ~3–5 per helper ratio is practical guidance, not a hard count. Apply it by walking the helper tree and asking: "If this helper throws, how many helpers does this verification case isolate it from?" A good verification case isolates one helper at a time for leaves, and one sub-tree at a time for integration cases.

When writing a new verification phase in a plan, list the helpers that phase covers and ensure the verification cases in that phase let you distinguish which helper failed — not just whether the phase passed or failed as a unit.

## Plan-authoring checklist

Before dispatching a plan to subagents, validate it against this checklist:

- [ ] Every task that adds functionality extracts a NEW helper. No task adds inline OQL to an existing helper's body.
- [ ] OnEvent's target shape is documented at the top of the plan and referenced in every task. Each task explicitly states what OnEvent body should look like after the task lands.
- [ ] `ProcessXBatch`'s target shape is documented and grows by exactly one `(call ...)` line per task.
- [ ] [how-to/develop-oql.md](../how-to/develop-oql.md) appears at the top of the mandatory-reading list with the emphatic framing block from the Hard Rule above. (This is structural, not a soft checklist item — see Hard Rule near the top of this doc.)
- [ ] [reference/clauses.md § Clause size and decomposition](../reference/clauses.md#clause-size-and-decomposition) is in the mandatory-reading list, AND the plan structure mirrors its discipline.
- [ ] The cargo-cult source is named and is **the post-refactor shape** (e.g., Strategist `b779ca8`), not the pre-refactor plan.
- [ ] **Any OQL code in plan tasks was verified against the V1 impl, not recalled from memory.** (See "Plan authorship discipline" section above.)
- [ ] **Verification cases mirror the helper tree at ~3–5 cases per helper.** Independent trees verified independently before combined. (See "Verification mirrors the helper tree" section above.)
- [ ] Pre-flight rules (same-symptom-twice → stop, three-failures → stop, no destructive ops without authorization) are in the plan, applied to every task.
- [ ] **If the step makes two or more sequential LLM calls:** estimated combined input token volume using the 1 KB ≈ 250 token rule of thumb. If the sum approaches the org TPM limit (30K for Anthropic Sonnet at MAB), applied model-split (Haiku/Sonnet) or reduced-second-call-message workaround. (See "LLM call budget analysis for multi-call steps" above.)

If a plan fails any of these, fix the plan before dispatching subagents. Plans are cheap to revise; refactor commits to fix shape mistakes are not.

## Related

- [reference/clauses.md § Clause size and decomposition](../reference/clauses.md#clause-size-and-decomposition) — the canonical OQL discipline this implements at the plan level.
- [reference/control-flow.md § Nesting](../reference/control-flow.md#nesting) — why nested `with-table-if`s are an anti-pattern, and the flat-sequential alternative.
- [how-to/develop-oql.md](../how-to/develop-oql.md) — the consolidated authoring + triage manual. This doc is the most important document for any OQL implementation work; every plan must surface it with emphatic framing.
- [omega-knowledge-base/contributor/explanation/agent-triage-discipline-enforcement.md](../../omega-knowledge-base/contributor/explanation/agent-triage-discipline-enforcement.md) — why Pre-flight STOP rules fail in practice despite appearing in plans/specs/prompts, and how to extend the orchestrator-review checklist above with a concrete Pre-flight rule scan to catch violations before accepting a subagent's "DONE" report.
- [explanation/oql-execution-model.md](oql-execution-model.md) — solution sets, terms-not-expressions, clause boundaries. Why small clauses isolate failures.
