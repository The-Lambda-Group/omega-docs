[< Home](../README.md) | [How-to](README.md)

# Write a superpowers plan

This is the procedural manual for authoring a superpowers plan or spec. Read it before drafting one. It tells you what mandatory grounding sections every plan and spec must contain, what each section must do, and how to fill it in correctly.

The companion explanation doc is [agent-step-plan-shape.md](../explanation/agent-step-plan-shape.md), which covers *why* plan structure dominates over read docs and how each task must shape its diff. This how-to covers the **non-task** sections at the top of every plan/spec — the grounding sections that keep the implementer from flying off the rails before they reach Task 0.0.

## The four required sections

Every superpowers plan and every superpowers spec MUST contain these four sections at the top, in this order, before the first task:

1. **Required Reading** — the docs the implementer must read before any code change.
2. **Pre-flight Rules** — the rules the implementer re-reads before EVERY push.
3. **Floundering Signals** — the symptoms that mean STOP, regardless of plan progress.
4. **Stop Conditions** — the hard limits that, once hit, end the trajectory and surface to the user.

A plan or spec missing any of these is malformed. The orchestrator should fail the plan back to the author before dispatching subagents.

The gold-standard reference is the LS plan: `~/Development/wttc/mab/docs/superpowers/plans/2026-04-30-mab-load-subjects.md`. Its top-of-file is the cargo-cult source for everything below.

---

## Why all four sections are load-bearing

Each section prevents a specific failure mode that has been observed multiple times in the Omega/MAB ecosystem:

- **Required Reading absent or de-emphasised** → the implementer pattern-matches OQL on training data and produces nested-with-table-if anti-patterns. Reference: MAB Copywriter V1 Phase 2 produced a 90-line OnEvent with nested `(and ...)` blocks; refactor commit `2137486` was required to fix shape after the fact (documented in [agent-step-plan-shape.md](../explanation/agent-step-plan-shape.md)).
- **Pre-flight Rules absent** → the implementer treats each push as independent and never re-anchors. Reference: MAB Researcher V1 ran a ~600k-token retry-loop because no rule said "same-symptom-twice → STOP."
- **Floundering Signals absent** → the implementer enters build-mode after a failure (adds helpers/captures/error-handling instead of probing). Reference: MAB Enumerate's ~7 socket-hangs in a single session, documented in MAB session-handoff lines 351–365.
- **Stop Conditions absent** → the implementer auto-rationalises long autonomous runs as "the plan said it might take a while." Reference: 2026-04-30 LS session, near-miss where an implementer was about to start a 30-minute operational drain because the plan didn't carve verification (≤10 iterations) from operational scope. Pre-flight rules 16–18 and stop conditions 26 were added to the LS plan to close this gap mid-session.

These are not theoretical. They have all happened. The four sections are the minimum viable set of guard rails.

---

## A. Required Reading

### What it is

A list of every doc the implementer must have read before any code change in the plan. Not "should consider" — must have read. The implementer's first task (Task 0.0) is to confirm this in chat.

### What it must contain

**Universal items (every plan):**

- **The plan's spec doc.** The plan and spec are paired; one cites the other. The plan must list its spec at the top of Required Reading.
- **[explanation/agent-step-plan-shape.md](../explanation/agent-step-plan-shape.md)** — the plan-task contract. Every implementer needs the shape rules.
- **The project's session-handoff doc** if one exists (e.g. `omega-knowledge-base/projects/<project>/reference/session-handoff.md`). This is where prior failure modes, current state, and live-data caveats live.
- **The most recently shipped impl in the same repo** as the cargo-cult source. Name the commit SHA — not "the file at HEAD," because HEAD drifts.

**OQL-specific (every plan that touches OQL implementation work):**

- **`omega-docs/how-to/develop-oql.md` — THE SINGLE MOST IMPORTANT DOC.** This MUST appear FIRST in Required Reading, with the emphatic block from the plan-author's contract:

  ```markdown
  ### THE SINGLE MOST IMPORTANT DOC — read this twice, internalize it before any code change

  - **`omega-docs/how-to/develop-oql.md`** — the consolidated authoring + triage manual.
    The "Operating manual (re-read every session)" 9-rule TLDR at the top is non-negotiable.
    Probe-don't-build, Result-as-inspection-channel, the verb shift for LLMs, two worked
    examples, wall-clock-as-cardinality-signal heuristic.

  This doc is more important than every other doc in this list combined.
  ```

  No substitutions. No de-emphasis. No burying it under other items. This is structural per [agent-step-plan-shape.md § Hard Rule](../explanation/agent-step-plan-shape.md).

- The OQL triage docs:
  - [how-to/narrow-an-oql-failure.md](narrow-an-oql-failure.md)
  - [how-to/triage-no-return.md](triage-no-return.md)
  - [how-to/debug-by-returning-early.md](debug-by-returning-early.md)
- **If the plan ends in live verification of a deployed impl, the task list and pre-flight rules must require one stored-entrypoint probe before any live LLM/write run.** The probe is: static scan that every helper in each `{"capture" [...]}` vector is defined earlier than the capturing clause, then `qo push` a production-shaped `OnEvent` with one downstream phase temporarily reduced to a constant payload, then `qo run` the stored impl through its real entrypoint. Copied scratch probes are not enough for this class.
- The OQL discipline references:
  - [reference/oql-hard-rules.md](../reference/oql-hard-rules.md)
  - [reference/control-flow.md](../reference/control-flow.md)
  - [reference/clauses.md](../reference/clauses.md) (especially § Clause size and decomposition AND § Closures / Capture)
  - [reference/reducers.md](../reference/reducers.md)
- The relevant gotchas — and the author MUST scan the `omega-docs/gotchas/` directory for any new entries since the last plan was written. Common ones to consider:
  - `gotchas/run-term-in-memory-clause.md`
  - `gotchas/with-table-if-capture-header.md`
  - `gotchas/lexical-scope.md`
  - `gotchas/conventions.md`
  - `gotchas/page-by-name-full-scan.md`
  - `gotchas/call-inside-with-table-if.md`
  - `gotchas/omega-cli-host-default.md`

**Domain-specific (e.g. MAB):**

- The relevant data-model doc(s). For MAB workflow steps: `wttc/mab/docs/reference/<workflow>-workflow-data-model.md`.
- The relevant protocols doc. For MAB: `wttc/mab/docs/reference/protocols.md`.
- The component-layout / library-layout pair (mandatory together for any plan that touches component definitions or installation — see [contributor/reference/superpowers-conventions.md § 3](../../omega-knowledge-base/contributor/reference/superpowers-conventions.md)).
- Cross-project canonical docs the project README points outward to (e.g. `omega-ai-components/docs/how-to/create-a-connector-component.md` for any plan that defines connectors).

### Cargo-cult template

Use this as starting text. Edit the bullets to match the plan's actual scope.

```markdown
## REQUIRED READING — read in full before Task 0.0

### THE SINGLE MOST IMPORTANT DOC — read this twice, internalize it before any code change

- **`omega-docs/how-to/develop-oql.md`** — the consolidated authoring + triage manual.
  The "Operating manual (re-read every session)" 9-rule TLDR at the top is non-negotiable.
  Probe-don't-build, Result-as-inspection-channel, the verb shift for LLMs, two worked
  examples, wall-clock-as-cardinality-signal heuristic.

This doc is more important than every other doc in this list combined. If you have not
internalized its iteration loop (one change per push, return early to debug, never batch
edits), you will reproduce the failure modes documented in this project's session-handoff.

### Required (canonical contracts)

- <project data-model doc> § <relevant section> — the design.
- <project protocols doc> § <relevant section>.
- <component-layout / library-layout pair if touching component definitions>.

### Required (failure triage + debug)

- `omega-docs/how-to/narrow-an-oql-failure.md`
- `omega-docs/how-to/debug-by-returning-early.md`
- `omega-docs/how-to/triage-no-return.md`

### Required (plan + write-shape discipline)

- `omega-docs/explanation/agent-step-plan-shape.md`
- `omega-docs/explanation/agent-step-write-shapes.md` (if the plan emits LLM-driven writes)
- `omega-docs/explanation/code-step-pagination-pattern.md` (if the plan paginates)

### Required (OQL discipline)

- `omega-docs/reference/oql-hard-rules.md`
- `omega-docs/reference/control-flow.md`
- `omega-docs/reference/clauses.md` § Clause size and decomposition, § Closures / Capture
- `omega-docs/reference/reducers.md`
- `omega-docs/gotchas/<each one in scope>.md`

### Required (engine bug workarounds — open gaps)

- `omega-knowledge-base/gaps.md` § <relevant gap entries>.

### Required (project + reference)

- `omega-knowledge-base/projects/<project>/reference/session-handoff.md` — entire file.
- **`<repo>/<path>/<most-recent-shipped-impl>.oql` at commit `<SHA>`** — primary cargo-cult source.

After reading: if any doc disagrees with the spec or plan, STOP and surface before proceeding.
```

### Project-specific extension points

- For MAB: add `wttc/mab/docs/reference/component-layout.md` and `library-layout.md` together; never one without the other.
- For OQL plans against `omega-db` or other engine-touching work: add the relevant `omega-db/docs/` pointers.
- For new agent-step plans: add the prior agent step's *post-refactor* impl as the cargo-cult source, not the pre-refactor one (per [agent-step-plan-shape.md § The trap](../explanation/agent-step-plan-shape.md)).

---

## B. Pre-flight Rules

### What it is

A numbered list of rules that the implementer re-reads before EVERY push. Not once. Every push. The list is the floor of discipline; project-specific rules append to it.

### What it must contain

**Universal rules (every plan):**

1. **Error → KB, not error → guess.** Re-anchor on the relevant doc before forming a hypothesis. (See [omega-knowledge-base/contributor/how-to/agent-error-recovery.md](../../omega-knowledge-base/contributor/how-to/agent-error-recovery.md) for the full discipline.)
2. **Walk back errors by Result-binding** (per [debug-by-returning-early.md](debug-by-returning-early.md)). The state IS the bug report.
3. **Inspect, don't infer.** Don't add helpers to "fix" — redirect Result and look.
4. **Same symptom twice → STOP, ask the user.**
5. **Three failed queries in a row → STOP**, no exceptions.
6. **No destructive ops** (`qo drop`, `qo delete-page`, `qo run install`) without explicit user authorization.
7. **Spec deviation requires user OK.**
8. **No skipping a failing verification case.**
9. **One change per push.**
10. **Evidence before assertion.**
11. **No hallucinated APIs, clauses, signatures, or arguments.** If the signature isn't in a doc you read this session, you haven't seen it.
12. **Result IS the print statement.** No new logging mechanism, no new error-handling layer — bind Result to what you want to see.
13. **Subagent dispatches use Sonnet by default.** Set `model: "sonnet"` explicitly on every Agent dispatch unless promotion to Opus is justified (per [agent-step-plan-shape.md § Subagent model selection](../explanation/agent-step-plan-shape.md)).
14. **Socket hang / engine timeout / `omega/jvm/Exception` → STOP.** Same-symptom-twice fires on FIRST occurrence in this category. DO NOT retry.
15. **Total expected wall-clock > 5 minutes → STOP, ASK USER.** Even if every individual call is fast. Even if a plan-text Expected says it's authorized. Multi-minute autonomous runs during development are almost always wrong — verification doesn't require exhaustive drains. Verify the loop on a handful of iterations (5–10), then ask whether the user wants the full operational run before continuing. **Do not auto-rationalize "the plan says 30 min, so it's fine."**
16. **Verification scope ≠ operational scope.** Verification = "does the loop work" (≤10 iterations, < 1 minute). Operational = "drain all N items end-to-end." If the plan asks for an operational-scale run as part of "verification," that is a plan defect — defer the operational portion, surface to the user, ask before doing it.
17. **Mid-task time-budget check.** If you find yourself thinking "this will take ~N minutes" and N > 5, STOP before kicking it off. Surface to the user.
18. **Before live verification of a deployed impl, prove the stored entrypoint with `qo run`.** Do a static capture-order scan, then push a production-shaped `OnEvent` with one downstream phase temporarily reduced to a constant payload and run it through the real stored entrypoint. Copied scratch probes do not test stored implementation capture resolution.

**Project-specific rules append after the universal list, numbered to continue.** Example from the LS plan: rules 19–29 would cover LS-specific concerns (provider returns empty on cold-start, K=40 per-arm `run-term` discipline, count-as-cursor invariant). Author appends similar rules per plan.

### Cargo-cult template

```markdown
## Pre-flight rules — read before EVERY push

Inherited verbatim from the universal Pre-flight Rules in
[omega-docs/how-to/write-a-superpowers-plan.md § B](../../omega-docs/how-to/write-a-superpowers-plan.md).
Project-specific extensions follow.

1. Error → KB, not error → guess.
2. Walk back errors by Result-binding.
3. Inspect, don't infer.
4. Same symptom twice → STOP.
5. Three failed queries → STOP.
6. No destructive ops without authorization.
7. Spec deviation requires user OK.
8. No skipping a failing verification case.
9. One change per push.
10. Evidence before assertion.
11. No hallucinated APIs/clauses/signatures.
12. Result IS the print statement.
13. Subagent dispatches use Sonnet (`model: "sonnet"` explicit).
14. Socket hang / engine timeout → STOP, same-symptom-twice on FIRST.
15. Total expected wall-clock > 5 min → STOP, ASK USER.
16. Verification scope ≠ operational scope.
17. Mid-task time-budget check (N > 5 min → STOP).
18. Before live verification of a deployed impl, prove the stored entrypoint with `qo run`.

<!-- project-specific rules continue here, numbered 19+ -->
19. <project-specific rule, e.g. "Provider returns no-return → Phase 0 audit re-runs.">
20. <...>
```

### Project-specific extension points

When adding rules 19+, follow the same shape: short imperative title, one-line rationale, what to do (STOP / surface / re-run audit / etc.). Stop conditions in the next section overlap with these — Pre-flight Rules are continuous (re-read every push), Stop Conditions fire once.

---

## C. Floundering Signals

### What it is

The symptoms that mean STOP, regardless of plan progress, even if no Pre-flight Rule has triggered. Floundering is when the implementer is making forward motion but the motion is wrong — adding helpers instead of probing, theorizing instead of inspecting, retrying without re-anchoring.

### What it must contain

**Universal floundering signals (every plan):**

- **Theorize-mode.** Writing "I think the issue is..." or "this is probably because..." before running a probe. (Per [develop-oql.md § Operating manual rule 5](develop-oql.md).)
- **Build-mode.** Adding helpers, captures, error-handling, or wrapper logic instead of redirecting Result and inspecting. (Per [develop-oql.md § Mindset: probe, don't build](develop-oql.md).)
- **Same symptom twice or three failures in a row.** Pre-flight rules 4 and 5 say STOP; the floundering signal is the moment you notice it.
- **Wall-clock spike** on a query that doesn't include a new HTTP/network call. Cardinality leak; do not add features, find the leak.
- **Socket close / ECONNRESET / engine timeout / `omega/jvm/Exception`.** Same-symptom-twice fires on FIRST occurrence.
- **Multi-minute autonomous runs without explicit authorization.** The hard rule on per-firing slowness is the floor, not the ceiling — total-task time is also a signal. If you find yourself running a query that will take minutes before reporting back, surface to the user instead.

### Cargo-cult template

```markdown
**Floundering signals — STOP when any fire:**
- Theorize-mode: writing "I think the issue is..." before a probe.
- Build-mode: adding helpers/captures/error-handling instead of probing.
- Same symptom twice or three failures in a row.
- Wall-clock spike on a query that doesn't include a new HTTP/network call.
- Socket close / ECONNRESET / engine timeout.
- **Long-task signal:** any plan step whose expected total wall-clock is measured in
  minutes (not seconds) before reporting back. Multi-minute autonomous runs need
  explicit authorization on the live data, not standing approval from plan text.
```

### Project-specific extension points

Append plan-specific floundering signals — e.g. "wall-clock for `<HelperX>` exceeds Ns at K=40" or "non-deterministic `prop-vals-by-folder` ordering surfaces in pagination cursor." Keep each signal a one-line symptom description; the response is always STOP.

---

## D. Stop Conditions

### What it is

Hard limits that, once hit, end the current trajectory and surface to the user. Pre-flight rules are continuous (re-read every push); Floundering Signals are subjective (notice the pattern); Stop Conditions are objective thresholds (this number, this state).

### What it must contain

**Universal hard stops (every plan):**

- **Per-query > 30s** for a known-fast category (or > 60s for a known-slow category — name the category in the plan).
- **Total task > 5 min wall-clock.** This is the same threshold as Pre-flight rule 15, expressed as a stop condition for objective tracking.
- **3 query failures in a row.**
- **Same error twice.**
- **Result shape divergence in a verification step.** The Result envelope produced does not match the envelope the plan documents — stop, do not "fix forward."
- **Destructive ops without authorization** (`qo drop`, `qo delete-page`, `qo run install`).
- **Spec disagreement.** Any doc in Required Reading disagrees with the plan or spec — surface.

### Cargo-cult template

```markdown
## Stop Conditions

Universal:

1. Per-query > 30s (or > 60s for known-slow categories: <name them>).
2. Total task wall-clock > 5 min — surface to user.
3. 3 query failures in a row.
4. Same error twice.
5. Result shape divergence in a verification step.
6. Destructive ops attempted without explicit user authorization.
7. Required-reading doc disagrees with spec/plan — surface.

Project-specific:

8. <plan-specific objective threshold, e.g. "Provider returns empty subjects on first
   cold-start with limit=3 + 150k-row pool — bisect via Result-binding.">
9. <...>
```

### Project-specific extension points

Each plan-specific stop condition should describe an objective state that, when observed, ends the trajectory. Examples from the LS plan:

- "Provider returns empty subjects on first cold-start with limit=3 + 150k-row pool — bisect via Result-binding."
- "Per-arm `run-term CallPlatformLoadSubjectsForArm` ECONNRESET — discriminator insufficient; surface."
- "`count(SA)` advances by something other than `WrittenCount` between firings — cursor invariant violated; surface."

Each is observable (the implementer can answer "yes/no" without judgement) and each terminates the trajectory.

---

## How the four sections compose

The sections layer:

```
Required Reading      → BEFORE Task 0.0     → "have I read this?"
Pre-flight Rules      → BEFORE every push   → "am I about to break a rule?"
Floundering Signals   → DURING work         → "am I in build/theorize-mode?"
Stop Conditions       → AT thresholds       → "have I hit a hard stop?"
```

A well-formed plan or spec contains all four. An implementer who reads all four at the start, re-reads the Pre-flight Rules before every push, watches for Floundering Signals during work, and surfaces on Stop Conditions, cannot fly off the rails — every degenerate trajectory (build-mode, retry-loops, unauthorized drains, hallucinated APIs) is caught by one of the four.

If you author a plan and find yourself thinking "this section feels redundant with the others," it isn't. The redundancy is intentional: each section catches a different class of failure at a different time in the loop.

---

## Plan vs spec — what differs

Plans and specs both need all four sections. The difference is granularity:

- **Spec** carries the full Required Reading list (the reference for the entire feature) and the full universal Pre-flight Rules + Floundering Signals + Stop Conditions. Project-specific extensions in the spec describe the design's known failure modes.
- **Plan** inherits Required Reading from the spec by reference (e.g. "REQUIRED READING — see spec § X, plus the additions below"), inherits Pre-flight Rules verbatim, and adds plan-specific Stop Conditions for the implementation work itself.

If a plan and spec disagree on any of the four sections, the plan wins for the implementation work (it's closer to the diff being produced) — but the disagreement is a defect to surface to the author.

---

## Review checklist

Before dispatching subagents to a plan, the orchestrator validates:

- [ ] Required Reading exists, lists develop-oql.md FIRST with the emphatic block (if OQL work).
- [ ] Required Reading lists the spec, the project session-handoff, and a named cargo-cult source with commit SHA.
- [ ] Pre-flight Rules section exists with at minimum the 17 universal rules.
- [ ] Floundering Signals section exists with at minimum the 6 universal signals.
- [ ] Stop Conditions section exists with at minimum the 7 universal stops.
- [ ] Project-specific rules / signals / stops are appended after each universal block, not interleaved (so the universal floor stays visible).
- [ ] The cargo-cult source named in Required Reading is the **post-refactor** shape, not the pre-refactor plan (per [agent-step-plan-shape.md § The trap](../explanation/agent-step-plan-shape.md)).

If any check fails, fix the plan before dispatching subagents. Plans are cheap to revise; recovering from a malformed plan mid-execution is not.

---

## Related

- [explanation/agent-step-plan-shape.md](../explanation/agent-step-plan-shape.md) — the per-task contract (one new helper + one `(call ...)` line + Result update). This how-to covers the non-task sections; that explanation covers task shape.
- [explanation/agent-step-write-shapes.md](../explanation/agent-step-write-shapes.md) — the three write-half patterns every workflow agent step must pick correctly.
- [explanation/code-step-pagination-pattern.md](../explanation/code-step-pagination-pattern.md) — canonical pagination shape for code steps.
- [how-to/develop-oql.md](develop-oql.md) — THE SINGLE MOST IMPORTANT DOC for any OQL implementation work. Required Reading on every OQL plan.
- [how-to/narrow-an-oql-failure.md](narrow-an-oql-failure.md) — OQL-specific triage discipline; minimum reproduction first.
- [omega-knowledge-base/contributor/reference/superpowers-conventions.md](../../omega-knowledge-base/contributor/reference/superpowers-conventions.md) — Omega/WTTC superpowers conventions: working artifacts (`docs/superpowers/` is gitignored), model selection (Sonnet default), mandatory-reading list curation (read project reference README first).
- [omega-knowledge-base/contributor/how-to/agent-error-recovery.md](../../omega-knowledge-base/contributor/how-to/agent-error-recovery.md) — the operational discipline behind Pre-flight Rule 1 (error → KB, not error → guess); the three failure modes (cheapest-hypothesis-untested, dispatcher-framing-inheritance, symptom-doc-without-mechanism-doc).
