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

## Plan-authoring checklist

Before dispatching a plan to subagents, validate it against this checklist:

- [ ] Every task that adds functionality extracts a NEW helper. No task adds inline OQL to an existing helper's body.
- [ ] OnEvent's target shape is documented at the top of the plan and referenced in every task. Each task explicitly states what OnEvent body should look like after the task lands.
- [ ] `ProcessXBatch`'s target shape is documented and grows by exactly one `(call ...)` line per task.
- [ ] [how-to/develop-oql.md](../how-to/develop-oql.md) appears at the top of the mandatory-reading list with the emphatic framing block from the Hard Rule above. (This is structural, not a soft checklist item — see Hard Rule near the top of this doc.)
- [ ] [reference/clauses.md § Clause size and decomposition](../reference/clauses.md#clause-size-and-decomposition) is in the mandatory-reading list, AND the plan structure mirrors its discipline.
- [ ] The cargo-cult source is named and is **the post-refactor shape** (e.g., Strategist `b779ca8`), not the pre-refactor plan.
- [ ] Pre-flight rules (same-symptom-twice → stop, three-failures → stop, no destructive ops without authorization) are in the plan, applied to every task.

If a plan fails any of these, fix the plan before dispatching subagents. Plans are cheap to revise; refactor commits to fix shape mistakes are not.

## Related

- [reference/clauses.md § Clause size and decomposition](../reference/clauses.md#clause-size-and-decomposition) — the canonical OQL discipline this implements at the plan level.
- [reference/control-flow.md § Nesting](../reference/control-flow.md#nesting) — why nested `with-table-if`s are an anti-pattern, and the flat-sequential alternative.
- [how-to/develop-oql.md](../how-to/develop-oql.md) — the consolidated authoring + triage manual. This doc is the most important document for any OQL implementation work; every plan must surface it with emphatic framing.
- [explanation/oql-execution-model.md](oql-execution-model.md) — solution sets, terms-not-expressions, clause boundaries. Why small clauses isolate failures.
