[< Home](../README.md) | [How-to](README.md)

# Develop OQL

This is the manual for working in OQL. Read it before touching a `.oql` file. It covers the mindset both authoring and debugging depend on, the inspection mechanic the language gives you in place of print statements, and the loop you operate forward (when adding) or backward (when debugging). Authoring and triage are the same loop in two directions; they share this doc.

## Operating manual (re-read every session)

These rules are what to actually do, in order. The rest of the doc is rationale. If you're an agent and you only have time for one thing, read this section.

1. **Each run answers exactly one question.** Before changing the file, name the question out loud (in chat or comment): "what's bound to X right now?", "does this fold collapse?", "what shape does write-table reject?" If you can't name the question, you're in build-mode. Stop and name it.
2. **Inspect by mutating `Result`.** Comment out everything after the term you want to look at. Bind `Result` to the variable, list, or map you want to surface. Run. Read the output. There are no print statements in OQL — `Result` IS the inspection channel.
3. **When a term fails, comment it out and bind `Result` to its inputs.** The failing term told you *that* it failed. Its inputs tell you *what data it failed on*. Don't theorize; look at the inputs. The bug is almost always visible there.
4. **When wall-clock grows past 1–2 seconds for a small workload, you have a cardinality leak.** Don't add features — find the leak. Common shapes: an unfolded `(get list _ var)` interacting with another `(get otherlist _ othervar)` (cross-product), a captured-helper invocation that opens N solutions when you thought it returned one, a `with-group-by` over many distinct keys.
5. **If you catch yourself writing "I think the issue is…" or "this is probably because…", stop.** That's the theorize-mode signal. Close the file, redirect `Result` to the relevant intermediate, run. The state will tell you. Theories are post-hoc and disposable; observations decide the next probe.
6. **Failures are data, not setbacks.** A failing run tells you the boundary. Sometimes you deliberately run something you expect to fail just to see *how* it fails. Don't gear-shift into recovery-mode when a probe fails — read the error, comment out the offending term, run the next probe.
7. **Iteration size matches your uncertainty, not a rule.** When you don't know what's happening, change one line and run. When you've just confirmed the previous probe and are laying down known-good structure, you can change five or ten lines. Bias toward small when in doubt.
8. **Killing the omega-cli process does NOT stop the server.** A hung query keeps running on the engine until it completes or times out. If you've launched a hang, don't launch another — wait for the engine to drain, or accept that the next several probes will be slow until it does.

If at any point you notice you're not following these — you're adding helpers/error-handling/captures to "fix" something, you're back-pedalling between named "Steps", you're proposing theories before running probes — stop, re-read this section, and pick the next probe.

## Mindset: probe, don't build

OQL development is not "write the program, run the tests." It is "operate the REPL — each run is a probe." Every change you make to a `.oql` file is the next command at a shell-style prompt. You're not constructing a program; you're typing the next question.

The shell analog is exact. You already do this fluently with bash:

- `git status` — what's the state of the tree?
- `qo describe` — what's the table schema?
- `omega-cli run-query <file>` — what is `Result` bound to right now?

Each run is one question. The next change to the file is the next question. Output flows back; you decide the next probe. The file is a scratchpad for the current command, not a program you're constructing.

Failures are answers too. They tell you where the dragons are. You're building a working map of the engine's behavior as you iterate; that map IS the work.

### The verb shift, especially for LLMs

When an LLM looks at a `.oql` file with `(:- ...)` clause definitions and a top-level `(return ...)`, it slides into "writing a Python program with print statements" mode. There are no print statements in OQL — there is only `Result`, and you inspect by mutating what `Result` is bound to. The verb is **look**, not **build**.

Symptoms of build-mode in OQL work:

- Adding helpers, captures, or error handling to fix a bug instead of redirecting `Result` to see what's happening.
- Theorizing about why a term failed instead of running a probe to inspect the state right before it.
- Treating a failure as a setback that needs a recovery plan, instead of a data point that narrows the next probe.
- Producing 20-line restructurings between runs instead of one-line redirections.
- Trying to "land" the next plan task per run, instead of asking "what's the question this run answers?"

When you catch yourself in any of these, gear-shift back to REPL mode. Treat the next change as a shell command, not a code edit.

## The inspection mechanic

OQL has no separate log channel. You don't `console.log` a value — you redirect `Result` to it. To see what's at any point in the program, do three things:

1. Comment out (or delete) every term after the point you want to inspect.
2. Bind `Result` to whatever you want to surface — a single variable, a list, a map of multiple variables.
3. Run.

```oql
;; want to see what TreatmentsDbId resolved to:
(call BuildCtx Exec Ctx)
(get Ctx "treatments-db-id" TreatmentsDbId)
(= Result TreatmentsDbId)
```

```oql
;; want to see all bindings up to a failure point:
(call BuildCtx Exec Ctx)
(call NewActiveTreaments Ctx NewIds)
(call ExistingActiveTreaments Ctx ExistingIds)
(length NewIds NewCount)
(length ExistingIds ExistingCount)
(= Result {"new-count" NewCount "existing-count" ExistingCount})
;; (call ComputeRetireSet ...) <-- commented out, this was failing
;; ...
```

When something errors unexpectedly, the move is the same: comment out the failing term and the rest of the body, bind `Result` to whatever was bound before it. That state IS the bug report — read it, decide the next probe.

You can also deliberately run a query that you expect to fail, just to see what fails. A failing run tells you the boundary; a working run tells you the inside. Both are data.

### Probe a failing term: comment it out and inspect its inputs

When a term throws or produces no-return, the cheap diagnosis is: comment it out and bind `Result` to the **arguments you were about to pass to it**. The error told you *that* the term failed; binding Result to its inputs tells you *what data it was failing on*. Those two pieces together usually identify the bug.

Suppose this fails with `omega/query/no-return`:

```oql
(call BuildCtx Exec Ctx)
(get Ctx "treatments-db-id" TreatmentsDbId)
(Qo.Public.OqlApi.Db.Prop/sec-index TreatmentsDbId "status" "active" PageId)
(Qo.Public.OqlApi.Db.Prop/prop-vals PageId PropVals)
(get PropVals "prop-vals" PV)
(= Result PV)
```

Stack frame points at `sec-index`. Don't theorize. Comment out from `sec-index` onward, bind `Result` to its inputs:

```oql
(call BuildCtx Exec Ctx)
(get Ctx "treatments-db-id" TreatmentsDbId)
;; (Qo.Public.OqlApi.Db.Prop/sec-index TreatmentsDbId "status" "active" PageId)   <-- failing term
;; (Qo.Public.OqlApi.Db.Prop/prop-vals PageId PropVals)
;; (get PropVals "prop-vals" PV)
(= Result {"TreatmentsDbId" TreatmentsDbId
           "looking-for-status" "active"})
```

Run. The output shows the inputs that produced the failure:

```json
{"TreatmentsDbId": "b386c731-...", "looking-for-status": "active"}
```

Now you can read the bug. `TreatmentsDbId` is bound to a real folder-id, so the lookup target is fine. Filtering for `status="active"` returned zero solutions — which means either (a) the table has no active rows, or (b) the sec-index isn't materialized for that key. Either way, the diagnosis is "no matching rows," not "the call is broken." The next probe runs `qo describe` and `qo query` against `b386c731-...` to confirm which.

The pattern generalises. If `(call SomeHelper A B C Out)` fails:

```oql
;; (call SomeHelper A B C Out)
(= Result {"A" A "B" B "C" C})
```

If `(write-table WriteInput Result)` fails:

```oql
;; (write-table WriteInput Result)
(= Result WriteInput)
```

The failed term's inputs are the next thing to read. You don't need a theory of why — the engine refused to do the operation on those inputs. Look at the inputs.

## Mental model: intermediate solution shapes

OQL is solution-table-oriented like SQL is row-oriented. At every term, hold a mental model of: how many solutions are in scope, what's their shape, and what each downstream term will do to that count.

It is not "minimize solutions everywhere" — that would suggest folding everything into one giant list, which is wrong. The right heuristic is: **be intelligent about intermediate tables**. Think SQL.

- What's the effective primary key of each intermediate?
- How big is the join between this intermediate and the next term's input?
- Where would a combinatorial explosion appear if I'm not careful?
- Does this fold reduce the join overhead I'm about to pay, or does it just hide a problem one term over?

Folds belong where they make subsequent terms cheap. An anti-join wants list-shape inputs, not stream-shape. A `(get list _ var)` opens N solutions; if you don't fold them down before another `(get otherlist _ othervar)`, you get N×M. That's the mistake. Folds are how you keep cardinality intentional, not how you make data "smaller."

Helper clauses (`(:- (Helper In Out) ...)`) are your other lever. A helper invoked via `(call Helper In Out)` runs internally, collapses to a single solution at its output, and returns one binding to the caller. This is how you keep the OnEvent body's solution table small even when the work inside the helper is multi-solution. The pattern that scales:

```oql
(:- (BuildCtx Exec Ctx)                         ;; one solution in, one out
    ;; ... page walk, build a Ctx map ...
    (= Ctx {...}))

(:- (NewActiveTreatments Ctx Ids)               ;; one in, one out
    (get Ctx "factors-db-id" FactorsDbId)
    (call BuildCrossProduct FactorsDbId List)
    (get List _ Tuple)
    (json-stringify Tuple Id)
    (fold [] Id append Ids))

(:- (OnEvent Exec Result {"capture" [BuildCtx NewActiveTreatments ...]})
    (call BuildCtx Exec Ctx)
    (call NewActiveTreatments Ctx Ids)
    ;; ... five lines of (call ...) ...
    (= Result ...))
```

OnEvent's body is a sequence of `(call ...)` lines. No raw iteration in OnEvent. Each call collapses to one solution at OnEvent's level. The iteration happens inside helpers where it's scoped to the helper's solution table.

Caveat: `(call ...)` doesn't always drop intermediate solutions perfectly. The engine has known cases where solution-table residue accumulates. If a sequence of helper calls runs slowly or hangs, that residue is suspect. `(run-term Helper ...)` is a stricter alternative that runs the helper in a fresh partition (its canonical use is replacing `with-group-by` for partition-before-fold scaling — see [reducers.md § Performance](../reference/reducers.md) and [run-term-in-memory-clause.md](../gotchas/run-term-in-memory-clause.md)). Use `call` by default; reach for `run-term` only when you've measured a residue problem.

### Wall-clock as a cardinality signal

The fastest way to tell whether your solution counts are sane is to time the run. Solution count and wall-clock are tightly correlated — if a query that should produce a small intermediate suddenly takes 30 seconds when it used to take half a second, you're paying for a cardinality you didn't expect somewhere. Often a missing fold, an accidental cross-product between two `(get list _ var)` calls in scope at the same time, or a captured-helper invocation that opens N solutions when you thought it returned one.

Easiest way to time:

```bash
time omega-cli run-query -h $OMEGA_URL impl/.../file.oql
```

Reference points (depend on engine load and data, but the orders of magnitude are stable):

- < 1s — single-digit to low-double-digit solutions in scope. Healthy.
- 1–5s — hundreds to low thousands. Often fine, sometimes a sign you're folding too late.
- 5–30s — tens of thousands. Almost always a cross-product or unfolded iteration you didn't intend.
- 30s+ or hang — combinatorial growth, or a known engine pathology (e.g. with-group-by over many distinct keys, see [reducers.md](../reference/reducers.md)). Don't wait it out — kill it and probe.

You don't need precise numbers. The signal is the *delta*: if probe N+1 takes 100× longer than probe N for the same logical workload, the change you made between them grew the solution table beyond what you intended. That delta is itself a probe — it tells you the next change is "where did the cardinality leak in?"

A subtle but important point: a hung omega-cli kill on the client does **not** stop the server. The query keeps running on the engine until it completes or the request times out, which can be a long time. Kill stresses the engine cumulatively if you keep launching new hung queries. When wall-clock starts climbing, treat the next probe as a chance to *reduce* cardinality first (add a fold, narrow a sec-index lookup, wrap an iteration in a helper) before adding any new logic.

## The push-run cycle

For top-level scratch queries (one-off inspection, not a deployed implementation):

```bash
omega-cli run-query -h $OMEGA_URL path/to/file.oql
```

The file ends with `(return [Result])`. The push happens at run time; there is no separate compile step. See [run-ad-hoc-oql-for-debugging.md](../../omega-cli/docs/how-to/run-ad-hoc-oql-for-debugging.md).

For a deployed implementation under iteration, the file registers clauses and OnEvent, and you dispatch with `qo run`:

```bash
qo push impl/.../file.oql && qo run "Component Installs/.../Page" -p "Event Handler" -m on-event -a '[{}]'
```

During heavy iteration on an OnEvent body, the cleanest setup is: comment out the `set-implementation-clauses` block at the bottom of the file and replace it with a direct call:

```oql
;; (= Clauses {"on-event" {"2" OnEvent}})
;; (Qo.Public.OqlApi.Impl/set-implementation-clauses ImplId Clauses Result)

(= Exec {"method" {"page-id" "<the-page-id>"}})
(call OnEvent Exec Result)
(return [Result])
```

Now `omega-cli run-query` exercises OnEvent directly without the `qo push` / `qo run` dispatch round-trip. Faster cycle, same semantics. Restore the `set-implementation-clauses` form when you're ready to deploy.

### Verification stubs: `(= Result {...})` not `(return [...])`

When stripping a clause body to confirm dispatch works, the correct stub is:

```oql
(:- (OnEvent Exec Result {"capture" [...]})
    (= Result {"status" "ok"}))
```

`(= Result Value)` is how a clause binds its output variable — it is the standard output form in every clause body, including OnEvent. `(return [...])` is **not** the clause output form. It is a top-level impl-script form that appears at the very end of the `.oql` file to register clauses and signal success to `qo push`. It has no meaning inside a clause body.

Using `(return [{"key" Output}])` inside a clause body does NOT bind `Result`. The symptom is:

```json
[{"omegadb/type": "symbol", "value-data": "Result"}]
```

`Result` is an unbound symbol — the clause never wrote to its output variable. Swap `(return [...])` for `(= Result {...})` and re-push.

The distinction:

| Location | Correct form | What it does |
|---|---|---|
| Inside a clause body (`(:- ...)`) | `(= Result Value)` | Binds the clause output variable |
| End of the impl file (top-level) | `(return [Result])` | Registers clauses, signals success to `qo push` |
| Top of a scratch query file | `(return [Result])` | Returns the scratch result to `qo run` |

> Every V1 OQL implementation uses `(= Result {...})` exclusively inside clause bodies. If you see `(return [...])` inside a `(:- ...)` block in a plan or spec example, that example is wrong — correct it before pushing.

### Safe restore: use a stub OnEvent between tasks, not the production OnEvent

**Rule:** when a plan task ends with "restore the production OnEvent body, push to confirm," use a **safe restore stub** instead of the full production body. Only restore the real production OnEvent when explicitly testing end-to-end behavior (typically the final phase of a phased plan — Phase H or I).

**Why this matters:** if the production OnEvent has side effects — LLM calls, `write-table` writes, Memory or Reports rows — every "push + run" during intermediate development tasks silently fires those side effects. In the Analyst agent, the production V1 OnEvent called `ProcessAnalyst`, which when the Platform Metrics table was non-empty fired a full LLM cycle that wrote Memory and Reports rows. Every between-task "push to confirm clean restore" during Phases A–G consumed Anthropic API budget and polluted the Memory table. No error appeared; no test failed. The cost was invisible until the budget was gone.

**The safe restore stub:**

```oql
(:- (OnEvent Exec Result {"capture" [ResolveAnalystCtx]})
    (call ResolveAnalystCtx Exec Ctx)
    (= Result {"has-more" "false" "status" "safe-restore"}))
```

This stub verifies:

1. The impl compiles and deploys without errors (`qo push` succeeds).
2. The install page resolves correctly — `ResolveAnalystCtx` walks the page hierarchy and if it resolves, the plumbing is right.
3. Zero LLM calls and zero DB writes occur.

**What to adapt per agent:** swap `ResolveAnalystCtx` for your agent's context-resolver helper (the first captured helper that builds `Ctx` from `Exec`). The result envelope can always be `{"has-more" "false" "status" "safe-restore"}` — the value is just a recognisable sentinel that confirms the stub ran.

**Only restore the full production OnEvent when the plan task explicitly says "test end-to-end behavior."** That is the one task where side effects are the point — you want the LLM to fire, the writes to land, and the output to be real. Every other restore-for-deploy checkpoint should use the safe stub.

## Operating the loop forward: authoring

Each iteration adds one verifiable piece. The loop:

1. **Decide what question this run answers.** Not "what feature does this run add" — what's the *one thing* you want to learn? Examples: "did the page walk resolve the right page-ids?", "does the cross-product produce 27 rows?", "what does CombinedList look like?", "does this write succeed?".

2. **Make the smallest change that surfaces the answer.** Often a single line: redirecting `Result` to the variable you want to see, or adding the next term and updating `Result` to expose its output.

3. **Run.**

4. **Read the output. Decide the next question.**

Specific moves:

### Start with the plumbing

Write the minimum query that proves the basic wiring works — page walk, table resolution, the entry call. Return the raw result. Run, inspect. Don't add anything else until the wiring's clean.

### Add one transformation at a time

Need to parse JSON? Add `(json-stringify Parsed RawBody)` and bind `Result` to `Parsed`. Run, inspect. Need the first element? Add `(get Parsed 0 First)` and rebind `Result` to `First`. Run, inspect. Never add two untested steps.

### Never rewrite the file

When something needs to change, edit only that thing. Don't rewrite from scratch — you'll lose working code and reintroduce bugs you already fixed. Use incremental edits on the last known-working version.

### Avoid side effects until the inputs are verified

Don't call `write-table` until you've bound `Result` to the exact map you'd pass to it and confirmed the shape is correct. The Result-as-dry-run is the cheap version of the side effect:

```oql
;; Dry-run: bind Result to the would-be write-table input and inspect.
(= Result {"file-id" FolderId "data" {"header" Header "rows" Rows}})
;; After verifying the shape, swap in:
;; (Qo.Public.OqlApi.Db.Prop/write-table {"file-id" FolderId "data" {...}} Result)
```

### One clause, one concern

If you need to stringify a field, get it, stringify it, set it back — three lines, return the result, verify. Don't combine stringify + write-table + count in one untested block.

## Operating the loop backward: triage

Triage is the same loop in subtractive direction. Authoring runs forward — add one step, verify. Triage runs backward — strip one step, verify. Both depend on the same tight feedback cycle. The same rules about probing apply with extra force, because failures are exactly when LLMs slide hardest into theorize-and-build mode.

Every rule below produces an **artifact**. A minimum-repro command, a ruled-in/out list, a written Observed/Hypothesis/Test trio. If you finish triage without writing anything concrete down, you ran on vibes and you're going to ship a non-fix.

### 1. Minimum reproduction is the first move, always

Before any theory of cause, strip the failing invocation to the smallest thing that still reproduces. If what you're running is larger than what reproduces, you have not narrowed.

**Artifact:** a one-screen scratch file or `qo run` command that reproduces the failure. Pin its exact output at the top of your working notes.

The move:

```bash
# Failing call
qo add-protocol /apps/foo/p-new ./proto-spec.json   # Mango throw

# Start stripping. Delete args, fields, or wrapper layers. Re-run.
# The goal is: smallest input, smallest program, same error.
```

If the stripped version stops failing, you stripped too much — add one thing back. That one thing is now a suspect. If it still fails, strip more. Keep going until further stripping makes the error go away. That boundary is where the culprit lives.

Until you have a minimum repro, you do not have a bug report to yourself. You have a story.

### 2. A stack frame locates, it does not identify

Especially in OQL. `with-table-if`, `if`, clause dispatch, and wrapper chains each inline multiple sub-calls into **one printed stack frame**. The frame is a postal address, not a suspect list. The call you see in the frame may not be the call that failed — the culprit can be any term inside the inlined body.

Default move after seeing a frame: **shrink it**, not read a cause from it. Move the suspected term into an isolated scratch file and see whether it still fails on its own.

**Artifact:** a scratch file that invokes only one candidate sub-call from the frame, run and checked.

Anti-pattern: "The stack says `with-table-if` at `Qo.Page.Bl/page-block`, therefore the problem is the `page-block` call I can see in the condition." The stack said nothing of the kind. The stack said "the error occurred somewhere inside the body of this `with-table-if`." That body is a tree.

### 3. Pattern-matching a prior bug is a hypothesis, not a diagnosis

Recognising "this looks like the Mango I fixed yesterday" is evidence of **where to look**, not **what to do**. A prior fix applies only after a minimal repro confirms the same class of failure. The anti-pattern has a name: **confabulating a cause backward from a fix you want to apply**.

**Artifact:** write the prior fix's signature and the new failure's signature in two columns. If any cell differs, the hypothesis does not yet hold; test it with a min repro before acting on it.

```
Prior fix (2026-04-23)          This failure (2026-04-24)
-----------------------         -----------------------
Mango on page-block             Mango on page-block
Downstream with-group-by        ??? not yet confirmed
Fix: restructured wrapper       Fix: ??? -- not chosen yet
```

If the right column is blank or speculative, you are not ready to edit. Go back to rule 1.

### 4. Triage the stack layers — ruled-in / ruled-out / unknown

Omega failures could live at any of these layers:

- **Shell env** — wrong `OMEGA_URL`, `QO_APP_ID`, `QO_WORKSPACE_ID`, auth state
- **Connection / host** — DB unreachable, wrong port, stale kube context
- **CLI query shape** — `qo` command building a malformed query (see [qo-query-empty-table-500](../../query-omega-cli/docs/explanation/qo-query-empty-table-500.md))
- **Public API clause** (`Qo.Public.Api.*`, `Qo.Public.OqlApi.*`) — signature mismatch, missing args
- **Wrapper clause** (`Qo.Page.Bl/*`, `Qo.Db.Prop/*`, other non-public) — body references that widen `static-rows`, downstream `with-group-by` pinning extra args
- **Engine routing / tdef-matches** — call resolves to fact-term fallback when a clause should have matched, or vice versa (see [clause-or-fact-term-resolution](../../omega-db/docs/explanation/clause-or-fact-term-resolution.md))
- **Datastore write-index** — call binds more args than the matching view's field set (see [page-block-pageindex-miss](../../omega-db/docs/explanation/page-block-pageindex-miss.md))
- **CouchDB view state** — stale ddoc, view not materialised, indexer behind
- **Cache** — term-def cache, clause cache, stale after deploy

Before proposing a fix, write each layer down and mark it ruled-in / ruled-out / unknown **with evidence**:

```
Layer                   Status     Evidence
---------------------   -------    -------------------------------
Shell env               Ruled out  OMEGA_URL correct, other `qo` calls work
Connection              Ruled out  Same session succeeds on different queries
CLI query shape         Unknown    Haven't inspected emitted OQL
Public API clause       Unknown    Sig looks right, haven't min-repro'd
Wrapper clause          Ruled in   Wrapper has `with-group-by [AppId PageId]`
Engine routing          Unknown
Write-index             Unknown    Need to check PageIndex fields
```

A fix proposal before this table is filled in is a guess.

### 5. "I don't know yet" is the preferred state

Premature diagnosis is the expensive mode. The cheapest mode is holding uncertainty explicitly and letting the next test narrow it.

**Artifact:** if asked "what's causing this?" before a min repro exists, the correct answer shape is:

> "I don't know what's causing this yet. The stack frame is at X, which inlines sub-calls A, B, and C. The next thing I'll test is a scratch file that isolates C, because C is the only one that touches the datastore under load. If C still fails alone, the culprit is inside C; if not, I'll isolate B next."

Wrong answer shapes:

> "This is the same Mango we saw yesterday — the `page-block` call in the condition. Moving the `get` fixes it." (No min repro, pattern-matched, confabulated cause.)

> "The engine's index matcher is wrong — we need to patch `term-table->index-args`." (See [fix-query-not-engine](../../omega-db/docs/explanation/fix-query-not-engine.md). Engine changes are not an agent's first move.)

Saying "I don't know yet, next test is X" costs nothing and protects you from shipping a non-fix.

### 6. Discipline decays under momentum

The failure most likely to get pattern-matched is the one that comes **right after a success**. You just fixed a Mango, the next thing throws a Mango, your brain (or model weights) supply the same fix. That is exactly when rules 1–4 apply MORE, not less.

**Artifact:** when you find yourself thinking "oh I know this one," write the two-column comparison from rule 3 **before** editing. If you can't fill it in without guessing, you don't know this one yet.

The canonical trap: you fixed a bug by moving a `(get Page "app-id" AppId)` out of a `with-table-if` condition. The next similar-looking Mango appears. You reach for the same fix. You didn't re-read the frame, didn't min-repro, didn't check whether the visible call is the actual culprit. The fix ships. The Mango persists. Now you're debugging your own non-fix on top of the original bug.

## What NOT to do

These are the LLM-specific failure modes the mindset above is designed to prevent. If you catch yourself doing any of these, gear-shift back.

- **Don't add code to fix a bug.** Redirect `Result` and look. The bug is in state you haven't seen yet.
- **Don't add error handling, validation, or guards "just in case."** That's build-mode. Probe-mode binds Result to the relevant intermediate and observes what's actually there.
- **Don't theorize about why a hang or error happened.** Comment out the suspected term, bind Result to what was bound before it, run. The state will tell you. Theory is post-hoc and disposable.
- **Don't write the entire query top-to-bottom and run once.** That works in batch languages with rich error messages. OQL gives you a 500 from the middle of a chain. You catch the issue in 30 seconds incrementally; you waste an afternoon if you batch.
- **Don't rewrite the file when feedback comes on one thing.** Edit only that thing. Don't lose working code to a wholesale rewrite.
- **Don't treat a failed run as a setback.** It's a data point. Read it, narrow, run again. Sometimes a deliberately-failing run is the cheapest way to bound a hypothesis.
- **Don't pattern-match prior bugs without a min repro.** That's discipline rule 3, repeated because LLMs violate it the most.

## Cross-references

### Inspection mechanics

- [Debug by returning early](debug-by-returning-early.md) — the recipe for binding `Result` to a tuple of intermediate variables to see per-solution state.
- [Write scratch queries](write-scratch-queries.md) — scratch queries need `(return [Result])` or they produce a bare 500. Relevant because min-repro scratch files fall into this category.
- [Run ad-hoc OQL for debugging](../../omega-cli/docs/how-to/run-ad-hoc-oql-for-debugging.md) — `omega-cli run-query` flow for inspection queries.

### Triage specifics

- [Triage `omega/query/no-return`](triage-no-return.md) — the no-return error has three modes (scratch-file missing return, dispatch-layer registration mismatch, body-level zero-solutions). Dispatch-layer and body-level zero-solutions share `:failed-at nil` and the same misleading "Scratch queries require..." envelope; walk the four dispatch checks first, then bisect the body if all pass.
- [Test at realistic batch sizes](test-at-realistic-batch-sizes.md) — some bugs only appear at non-trivial batch sizes. If your min repro at `limit=1` works but the full run fails, scale the repro up before concluding anything.
- [Drop rows with forced unification failure](drop-rows-with-forced-unification-failure.md) and [Handle HTTP 422 in dispatch](handle-http-422-in-dispatch.md) — common shapes of `WRITING_SYMBOLS` triage.

### Reference docs to read before authoring

- [OQL hard rules](../reference/oql-hard-rules.md) — non-negotiable rules on forbidden datastores, forbidden operations, the boolean-string requirement. Read first.
- [Execution model](../explanation/oql-execution-model.md) — how OQL evaluates queries (solution sets, why terms can't be nested inside literals).
- [Built-in terms](../reference/built-ins.md) — the full built-in term reference.
- [Control flow](../reference/control-flow.md) — read before writing any `when`, `when-not`, `if`, or `with-table-if`.
- [Reducers](../reference/reducers.md) — `fold`, `with-count`, `with-group-by`, the fold-over-zero-solutions canonical pattern, when `run-term` is and isn't appropriate.
- [Closures and capture](../reference/clauses.md) — how `{"capture" [...]}` works on clause definitions, and the `call` vs `run-term` rule.

### Engine-side context when the failure is a Mango

- [No Mango by default](../../omega-db/docs/explanation/no-mango-by-default.md) — what `omega/query/full-scan` means, the three preferred fixes, why the bookend isn't a resolution.
- [Fix the query, not the engine](../../omega-db/docs/explanation/fix-query-not-engine.md) — the escalation order. Engine changes are never an agent's first move.
- [page-block misses PageIndex](../../omega-db/docs/explanation/page-block-pageindex-miss.md) — the canonical case study of a downstream reference widening `static-rows`.
- [Clause or fact-term resolution](../../omega-db/docs/explanation/clause-or-fact-term-resolution.md) — dispatch order and why a wrapper-clause-only datastore can still appear to Mango.
- [Fold partition scaling](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/explanation/fold-partition-scaling.md) — when a query is slow-but-correct and reducers are the suspect.

## Worked example — authoring loop in action

A six-probe walkthrough of building the Enumerate OnEvent body. Each probe shows: the **question** the run answers, the **change** to the file (usually one or two lines), the **run output**, and the **decision** about the next question. The whole loop ran in seconds; the work is the sequence of questions, not the lines of code.

The setup at the bottom of the file stays constant throughout — direct call to OnEvent, return the bound Result:

```oql
(= Exec {"method" {"page-id" "<enum-page-id>"}})
(call OnEvent Exec Result)
(return [Result])
```

Run with `omega-cli run-query -h $OMEGA_URL impl/.../enumerate.oql` between every probe.

### Probe 1: did the page walk resolve correctly?

**Change** (everything after the page walk replaced with a Result binding):

```oql
(= Result {"PageId" PageId
           "FactorsDbId" FactorsDbId
           "TreatmentsDbId" TreatmentsDbId})
```

**Output:**

```json
{"PageId": "4b54bd1b-...", "FactorsDbId": "d8b7f4a4-...", "TreatmentsDbId": "b386c731-..."}
```

**Decision:** all three IDs bound, walk is fine. Next question: does `BuildCrossProduct` produce the expected 27 tuples (3 factors × 3 levels each)?

### Probe 2: cross-product cardinality

**Change** (add the call, take the length, bind Result to the count):

```oql
(call BuildCrossProduct FactorsDbId CombinedList)
(length CombinedList CrossProductCount)
(= Result CrossProductCount)
```

**Output:**

```
27
```

**Decision:** matches expected. Next question: what does one tuple look like? Do we have the shape right for the next step?

### Probe 3: tuple shape

**Change** (index into the list, bind Result to the first element):

```oql
(get CombinedList 0 FirstTuple)
(= Result FirstTuple)
```

**Output:**

```json
[["credibility-source","founder-track-record"],["subject-tone","insider-insight"],["value-prop","wealth-building"]]
```

**Decision:** list of three `[factor-id, level]` pairs. `(json-stringify Tuple Id)` will produce the canonical treatment_id. Next question: does the helper that maps the tuple list to the id list work?

### Probe 4: target-id list via helper

**Change** (declare a helper, call it, bind Result to its output):

```oql
(:- (TupleListToTreatmentIds Tuples Ids)
    (get Tuples _ Tuple)
    (json-stringify Tuple Id)
    (fold [] Id append Ids))

;; ... in OnEvent ...
(call TupleListToTreatmentIds CombinedList TargetIds)
(= Result TargetIds)
```

**Output:**

```json
["[[\"credibility-source\",\"founder-track-record\"],[\"subject-tone\",\"insider-insight\"],...]",
 "[[\"credibility-source\",\"marc-kw-president\"],...]",
 ...27 strings...]
```

**Decision:** 27 JSON strings, the canonical treatment_ids. Next question: bind Result to the would-be `write-table` input to verify the shape before committing to the side effect.

### Probe 5: dry-run the write input

**Change** (build the rows, bind Result to the WriteInput map — no `write-table` yet):

```oql
(call BuildActiveRows TargetIds TargetRows)
(= WriteInput {"file-id" TreatmentsDbId
               "data" {"header" ["treatment_id" "status"]
                       "rows" TargetRows}})
(= Result WriteInput)
```

**Output:**

```json
{"file-id": "b386c731-...",
 "data": {"header": ["treatment_id", "status"],
          "rows": [["[[\"credibility-source\",...]]", "active"], ...27 rows...]}}
```

**Decision:** shape looks right — file-id resolves, header is two columns, 27 rows each `[treatment_id, "active"]`. Safe to swap in the actual write call.

### Probe 6: the actual write

**Change** (replace `(= Result WriteInput)` with the write-table call):

```oql
(Qo.Public.OqlApi.Db.Prop/write-table WriteInput WriteResult)
(= Result WriteResult)
```

**Output:**

```
"create"
```

**Decision:** write succeeded — 27 rows created. Side effect verified. The OnEvent body now does what we want; the rest is layering on retire-set logic via more probes.

### What the example shows

- Each change was small — a Result rebinding, a helper add, a single new line.
- Each run answered one specific question.
- Result was the inspection channel throughout. The body kept changing shape because `Result` was being redirected to whatever we wanted to look at this run.
- The actual side effect (`write-table`) was the **last** thing we ran. Up to probe 5, every run was free of consequences — just inspection. The dry-run pattern at probe 5 caught any shape mistake before the side effect committed.
- Total wall clock: about a minute. Total commits if we'd been committing each run: six.

Compare to writing all six probes' worth of code in one shot and running once at the end. If anything failed, the error would be a 500 from the middle of the chain with no map of where the data went wrong. Probing instead gives you a 30-second answer per question and a precise frontier between "known working" and "next thing to verify."

## Worked example — 2026-04-24 `qo add-protocol` (triage loop in action)

A real failure where the triage rules were misapplied first and applied second.

### The failure

```bash
qo add-protocol /apps/smartlead/p-new ./proto-spec.json
# -> OQLException: omega/query/full-scan
# -> functor=page-block/4, args.0 + args.1 bound
```

Stack frame: `Qo.Public.OqlApi.Proto/add-or-get-by-name!`, pointing at a `with-table-if` body.

### What an agent did wrong

1. Read the stack frame, saw the `with-table-if` body.
2. Noticed a visible `(Qo.Page.Bl/page-block _ PageId _ Block)` in the condition.
3. Pattern-matched to a prior fix ("AppId was getting pinned — we moved the `get` into the else branch").
4. Moved `(get Page "app-id" AppId)` into the else branch.
5. Re-pushed. Re-ran. **Same Mango.**

Rules violated: **(1)** no min repro before editing, **(2)** read the frame as a diagnosis instead of an address, **(3)** pattern-matched without filling in the comparison, **(6)** discipline decayed because the previous fix-class had been successful.

### What actually found the culprit

User wrote a stripped-down scratch file:

```oql
;; min repro -- construct the proto page, call append-last, return early
(Qo.Page/new-page AppId ParentPageId Name ProtoPage)
(Qo.Page.Bl/append-last ProtoPage ProtoBlock Ignored)
(= Result [ProtoPage ProtoBlock Ignored])
(return [Result])
```

Mango fired on this scratch. The `add-or-get-by-name!` `with-table-if` was irrelevant — the culprit was in the **else branch's** `append-last` chain:

```
(Qo.Page.Bl/append-last Page ProtoBlock _)
  -> append-last-rank
  -> max-rank!
  -> (Qo.Page.Bl/page-block AppId PageId BlockId Block)   ; <-- HERE
```

`max-rank!`'s body has a `page-block` call with **both** AppId and PageId bound. That is the off-index call that Mangoes. It has nothing to do with the `page-block` call visible in the `with-table-if` condition. Different site, different call, different args.

### The narrowing move

Shrinking the frame (rule 2) plus a min repro (rule 1) localised the culprit to `max-rank!`. Once localised, the fix was a wrapper restructure (per [fix-query-not-engine](../../omega-db/docs/explanation/fix-query-not-engine.md)) at `max-rank!`, not at `add-or-get-by-name!`.

### The lesson, stated as a rule

**Shrink the frame, do not read the cause from it.** A `with-table-if` frame names a tree. The culprit can be several levels deep in a branch that isn't visually adjacent to the stack pointer. Only a min repro tells you which leaf is at fault.
