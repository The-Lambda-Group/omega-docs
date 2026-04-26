[< Home](../README.md) | [How-to](README.md)

# Narrow an OQL failure

This is the triage mirror of [develop-oql-implementations.md](develop-oql-implementations.md). Authoring runs the push-run loop **forward** -- add one step, verify, add the next. Triage runs the same loop **backward** -- strip one step, verify, strip again. Both depend on the same tight feedback cycle. If you are debugging an OQL failure (Mango throw, no-return, unexpected result, `WRITING_SYMBOLS`, bare 500), you are the authoring loop in reverse -- read that doc first if you haven't, then come back.

Every rule below produces an **artifact**. A minimum-repro command, a ruled-in/out list, a written Observed/Hypothesis/Test trio. If you finish triage without writing anything concrete down, you ran on vibes and you are going to ship a non-fix.

## The six rules

### 1. Minimum reproduction is the first move, always

Before any theory of cause, strip the failing invocation to the smallest thing that still reproduces. If what you are running is larger than what reproduces, you have not narrowed.

**Artifact:** a one-screen scratch file or `qo run` command that reproduces the failure. Pin its exact output at the top of your working notes.

The move looks like:

```bash
# Failing call
qo add-protocol /apps/foo/p-new ./proto-spec.json   # Mango throw

# Start stripping. Delete args, fields, or wrapper layers. Re-run.
# The goal is: smallest input, smallest program, same error.
```

If the stripped version stops failing, you stripped too much -- add one thing back. That one thing is now a suspect. If it still fails, strip more. Keep going until further stripping makes the error go away. That boundary is where the culprit lives.

Until you have a minimum repro, you do not have a bug report to yourself. You have a story.

### 2. A stack frame locates, it does not identify

Especially in OQL. `with-table-if`, `if`, clause dispatch, and wrapper chains each inline multiple sub-calls into **one printed stack frame**. The frame is a postal address, not a suspect list. The call you see in the frame may not be the call that failed -- the culprit can be any term inside the inlined body.

Default move after seeing a frame: **shrink it**, not read a cause from it. Move the suspected term into an isolated scratch file and see whether it still fails on its own.

**Artifact:** a scratch file that invokes only one candidate sub-call from the frame, run and checked.

Anti-pattern to name: "The stack says `with-table-if` at `Qo.Page.Bl/page-block`, therefore the problem is the `page-block` call I can see in the condition." The stack said nothing of the kind. The stack said "the error occurred somewhere inside the body of this `with-table-if`." That body is a tree.

### 3. Pattern-matching a prior bug is a hypothesis, not a diagnosis

Recognising "this looks like the Mango I fixed yesterday" is evidence of **where to look**, not **what to do**. A prior fix applies only after a minimal repro confirms the same class of failure. The anti-pattern has a name: **confabulating a cause backward from a fix you want to apply**.

**Artifact:** write the prior fix's signature and the new failure's signature in two columns. If any cell differs, the hypothesis does not yet hold; test it with a min repro before acting on it.

```
Prior fix (2026-04-23)          This failure (2026-04-24)
-----------------------         -----------------------
Mango on page-block             Mango on page-block
Downstream with-group-by [AppId] pinned AppId   ??? not yet confirmed
Fix: restructured wrapper to drop AppId          Fix: ??? -- not chosen yet
```

If the right column is blank or speculative, you are not ready to edit. Go back to rule 1.

### 4. Triage the stack layers -- ruled-in / ruled-out / unknown

Omega failures could live at any of these layers:

- **Shell env** -- wrong `OMEGA_URL`, `QO_APP_ID`, `QO_WORKSPACE_ID`, auth state
- **Connection / host** -- DB unreachable, wrong port, stale kube context
- **CLI query shape** -- `qo` command building a malformed query (see [qo-query-empty-table-500](../../query-omega-cli/docs/explanation/qo-query-empty-table-500.md))
- **Public API clause** (`Qo.Public.Api.*`, `Qo.Public.OqlApi.*`) -- signature mismatch, missing args
- **Wrapper clause** (`Qo.Page.Bl/*`, `Qo.Db.Prop/*`, other non-public) -- body references that widen `static-rows`, downstream `with-group-by` pinning extra args
- **Engine routing / tdef-matches** -- call resolves to fact-term fallback when a clause should have matched, or vice versa (see [clause-or-fact-term-resolution](../../omega-db/docs/explanation/clause-or-fact-term-resolution.md))
- **Datastore write-index** -- call binds more args than the matching view's field set (see [page-block-pageindex-miss](../../omega-db/docs/explanation/page-block-pageindex-miss.md))
- **CouchDB view state** -- stale ddoc, view not materialised, indexer behind
- **Cache** -- term-def cache, clause cache, stale after deploy

Before proposing a fix, write each layer down and mark it ruled-in / ruled-out / unknown **with evidence**:

```
Layer                       Status     Evidence
---------------------       -------    -------------------------------
Shell env                   Ruled out  OMEGA_URL correct, other `qo` calls work
Connection                  Ruled out  Same session succeeds on different queries
CLI query shape             Unknown    Haven't inspected emitted OQL
Public API clause           Unknown    Sig looks right, haven't min-repro'd
Wrapper clause              Ruled in   Wrapper has `with-group-by [AppId PageId]`
Engine routing              Unknown
Write-index                 Unknown    Need to check PageIndex fields
```

A fix proposal before this table is filled in is a guess.

### 5. "I don't know yet" is the preferred state

Premature diagnosis is the expensive mode. The cheapest mode is holding uncertainty explicitly and letting the next test narrow it.

**Artifact:** if asked "what's causing this?" before a min repro exists, the correct answer shape is:

> "I don't know what's causing this yet. The stack frame is at X, which inlines sub-calls A, B, and C. The next thing I'll test is a scratch file that isolates C, because C is the only one that touches the datastore under load. If C still fails alone, the culprit is inside C; if not, I'll isolate B next."

Wrong answer shapes:

> "This is the same Mango we saw yesterday -- the `page-block` call in the condition. Moving the `get` fixes it." (No min repro, pattern-matched, confabulated cause.)

> "The engine's index matcher is wrong -- we need to patch `term-table->index-args`." (See [fix-query-not-engine](../../omega-db/docs/explanation/fix-query-not-engine.md). Engine changes are not an agent's first move.)

Saying "I don't know yet, next test is X" costs nothing and protects you from shipping a non-fix.

### 6. Discipline decays under momentum

The failure most likely to get pattern-matched is the one that comes **right after a success**. You just fixed a Mango, the next thing throws a Mango, your brain (or model weights) supply the same fix. That is exactly when rules 1-4 apply MORE, not less.

**Artifact:** when you find yourself thinking "oh I know this one," write the two-column comparison from rule 3 **before** editing. If you cannot fill it in without guessing, you do not know this one yet.

The canonical trap: you fixed a bug by moving a `(get Page "app-id" AppId)` out of a `with-table-if` condition. The next similar-looking Mango appears. You reach for the same fix. You did not re-read the frame, did not min-repro, did not check whether the visible call is the actual culprit. The fix ships. The Mango persists. Now you are debugging your own non-fix on top of the original bug.

## Worked example -- 2026-04-24 `qo add-protocol`

Concrete walkthrough of the rules applied (and mis-applied) on a real failure.

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
3. Pattern-matched to a prior fix ("AppId was getting pinned -- we moved the `get` into the else branch").
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

Mango fired on this scratch. The `add-or-get-by-name!` `with-table-if` was irrelevant -- the culprit was in the **else branch's** `append-last` chain:

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

**Shrink the frame, do not read the cause from it.** A `with-table-if` frame names a tree. The culprit can be several levels deep in a branch that is not visually adjacent to the stack pointer. Only a min repro tells you which leaf is at fault.

## Cross-references

### Authoring mirror

- [Develop OQL implementations](develop-oql-implementations.md) -- this doc's forward counterpart. Read it to internalise the push-run loop in the additive direction. Triage is the same loop in subtractive direction.

### Supporting triage mechanics

- [Debug by returning early](debug-by-returning-early.md) -- how to bind `Result` to a tuple of intermediate variables to see per-solution state. Use when the min repro reproduces but you need to see what the data looks like mid-pipeline.
- [Write scratch queries](write-scratch-queries.md) -- scratch queries need `(return [Result])` or they produce a bare 500. Relevant because min-repro scratch files fall into this category.
- [Triage `omega/query/no-return`](triage-no-return.md) -- the no-return error has three modes (scratch-file missing return, dispatch-layer registration mismatch, body-level zero-solutions). Dispatch-layer and body-level zero-solutions share `:failed-at nil` and the same misleading "Scratch queries require..." envelope; walk the four dispatch checks first, then bisect the body if all pass. Read this if you hit no-return on a dispatched call (`qo run` against a stored impl) -- adding `(return [...])` will not help because the impl is a clause, not a scratch file.
- [Test at realistic batch sizes](test-at-realistic-batch-sizes.md) -- some bugs only appear at non-trivial batch sizes. If your min repro at `limit=1` works but the full run fails, scale the repro up before concluding anything.
- [Drop rows with forced unification failure](drop-rows-with-forced-unification-failure.md) and [Handle HTTP 422 in dispatch](handle-http-422-in-dispatch.md) -- common shapes of `WRITING_SYMBOLS` triage.

### Engine-side context when the failure is a Mango

- [No Mango by default](../../omega-db/docs/explanation/no-mango-by-default.md) -- what `omega/query/full-scan` means, the three preferred fixes (write-index, primary-key path, bookend), and why the bookend is not a resolution.
- [Fix the query, not the engine](../../omega-db/docs/explanation/fix-query-not-engine.md) -- the escalation order. Engine changes are never an agent's first move.
- [page-block misses PageIndex](../../omega-db/docs/explanation/page-block-pageindex-miss.md) -- the canonical case study of a downstream reference widening `static-rows`.
- [Clause or fact-term resolution](../../omega-db/docs/explanation/clause-or-fact-term-resolution.md) -- dispatch order and why a wrapper-clause-only datastore can still appear to Mango.
- [Fold partition scaling](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/explanation/fold-partition-scaling.md) -- when a query is slow-but-correct and reducers are the suspect.
