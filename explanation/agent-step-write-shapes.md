[< Home](../README.md) | [Explanation](README.md)

# Agent-step write shapes

A workflow agent step (Manager, Researcher, Strategist, Copywriter, Critic, Analyst, Execute, …) finishes by writing batched results — a set of rows into a primary table plus a single Memory row recording the agent's reasoning. Four patterns recur in those write paths and have to be chosen correctly per agent. Picking the wrong pattern produces silent corruption (the wrong rows survive a uniqueness check), per-row identity collisions (`KEY_UPDATE_CONFLICT` on the very first agent that runs against pre-existing data), cold-start no-returns (the agent silently does nothing on its first deployment because its Memory guard fires against another agent's pre-existing rows), or silent mis-sourcing (a verification-role LLM summarises from prior verdicts instead of re-reading source data, passing checks it never actually ran).

This doc explains the four shapes side by side, the conditions that pick each one, and the failure modes if you cargo-cult the wrong shape into the next agent step. The patterns generalise to any workflow agent step that batches LLM output to a primary table and records a Memory row.

The canonical exemplars are MAB Manager (validation pattern), MAB Strategist (HTML-block write), MAB Copywriter (text-column write), and MAB Analyst Format-phase (LLM audit specificity). All four are flat-helpers shape per [agent-step-plan-shape.md](agent-step-plan-shape.md).

## Pattern 1 — uniqueness validation against a source array, not a folded accumulator

**Discovery context.** Manager validates LLM-produced `Commands` for duplicate `agent` values before any write. Strategist needs the same shape for unique `(treatment_id, name)` keys across drafts. The natural-feeling check is "fold the keys, count the fold output, throw if count is wrong." That check is **silently wrong** — it deduplicates the very thing it is trying to detect.

### Why fold collapses duplicates before you can count them

OQL solution sets are sets. A `fold` reduces over **unique values** of its input symbol — duplicate solutions are collapsed before they reach the reducer. A check like:

```oql
;; WRONG — AllKeys is already deduplicated by the time you measure its length.
(get Drafts _ Draft)
(get Draft "treatment_id" Tid)
(get Draft "name" Name)
(string-concat Tid "::" Partial)
(string-concat Partial Name PairKey)
(fold [] PairKey append AllKeys)
(length AllKeys Total)              ;; Total = unique count, not row count
(call IntoSet AllKeys UniqueSet)
;; ...compare Total vs unique set count → duplicates always pass.
```

`Total` is already the unique-key count. Comparing it to the unique-set count proves nothing — they are the same number by construction.

### The fix — count the source array directly

`(length Drafts Total)` measures the input array, before any fold runs. The fold then produces the deduplicated key set and the comparison is meaningful:

```oql
(:- (ValidateDraftNames Drafts Done {"capture" [IntoSet ThrowDraftNameCollision]})
    (get Drafts _ Draft)
    (get Draft "treatment_id" Tid)
    (get Draft "name" Name)
    (string-concat Tid "::" Partial)
    (string-concat Partial Name PairKey)
    (fold [] PairKey append AllKeys)
    (length Drafts Total)                 ;; ← source-array length, NOT (length AllKeys ...)
    (call IntoSet AllKeys UniqueSet)
    (get UniqueSet UKey _)
    (fold [] UKey append UniqueKeys)
    (length UniqueKeys UniqueCount)
    (with-table-if (= UniqueCount Total)
      [Total UniqueCount AllKeys UniqueKeys ThrowDraftNameCollision Done]
      (= Done "ok")
      (call ThrowDraftNameCollision AllKeys UniqueKeys Total UniqueCount Done)))
```

Manager's `ValidateCommands` mirrors the shape: `(length Commands TotalCount)` (source array) is compared to `(length UniqueAgents UniqueCount)` (post-fold), and `>=` flags duplicates.

The reference home for this rule is [reducers.md § fold folds over unique values, not rows](../reference/reducers.md#fold-folds-over-unique-values-not-rows) — including a "Counting source-array length vs. fold-accumulator length" callout. This explanation doc names the pattern; the reference doc names the primitive.

### Generalising

Any agent step that asks "did the LLM emit duplicates?" must measure pre-fold cardinality with `(length SourceArray Total)` and compare against post-fold cardinality. The shape is identical for `(treatment_id, name)` pairs (Copywriter), `agent` values (Manager), or any future per-row uniqueness invariant (Critic / Analyst review-IDs, Execute send-IDs).

## Pattern 2 — text-column write vs HTML-block write

**Discovery context.** Strategist writes one `Strategy` row per treatment with a rich-HTML Content block. Copywriter writes 2-4 `Copy` rows per treatment with a plain-text `content` column. Both share a treatment-id-keyed PK shape and both batch their LLM output into a primary table. They use **opposite** write shapes because the data shape is opposite.

### The two shapes

#### Shape A — single `write-table` to a text column (Copywriter)

The PK alone (`[treatment_id, draft]`) uniquely identifies the row. The `content` column is plain text. One `write-table` call writes every draft for the batch:

```oql
(:- (WriteAllDrafts CopyDbId Drafts WriteCount {"capture" []})
    (get Drafts _ Draft)
    (get Draft "treatment_id" WTid)
    (get Draft "name" WName)
    (get Draft "content" WContent)
    (= Row [WTid WName WContent])
    (fold [] Row append DraftRows)
    (length DraftRows WriteCount)
    (= WriteInput {"file-id" CopyDbId
                   "data" {"header" ["treatment_id" "draft" "content"]
                           "rows" DraftRows}})
    (Qo.Public.OqlApi.Db.Prop/write-table WriteInput _))
```

No per-row Content block. No `add-or-get-by-name`. No `run-term`. One batched write, one engine round-trip, full-table commit.

#### Shape B — `write-table` followed by per-row Content HTML block writes (Strategist)

The PK identifies the row but the strategy is rich HTML rendered into an `HtmlPageBlock` named `"Content"` on the row's page. The block is per-row and must live under per-row Block-UUID identity. The shape is:

```oql
;; Per-row helper. Invoked via run-term so each row gets a fresh solution.
(:- (WriteStrategy StrategiesDbId Tid Html Done)
    (Qo.Public.OqlApi.Db.Prop/row-index StrategiesDbId [Tid] RowPageId)
    (Qo.Public.OqlApi.Page/page-by-id RowPageId RowPage)
    (Qo.Public.OqlApi.Page.Bl.Html/add-or-get-by-name RowPage "Content" _)
    (Qo.Public.OqlApi.Page.Bl.Html/set-content RowPage "Content" Html _)
    (= Done "ok"))

;; Batch-write the rows, then per-row Content block via run-term.
(:- (WriteAllStrategies Ctx StrategiesMap StrategyCount {"capture" [WriteStrategy]})
    (get Ctx "strategies-db-id" StrategiesDbId)
    (get StrategiesMap RowKey _)
    (= RowEntry [RowKey])
    (fold [] RowEntry append StrategyRows)
    (length StrategyRows StrategyCount)
    (= StrategiesWriteInput {"file-id" StrategiesDbId
                             "data" {"header" ["treatment_id"]
                                     "rows" StrategyRows}})
    (Qo.Public.OqlApi.Db.Prop/write-table StrategiesWriteInput _)
    (get StrategiesMap WriteTid WriteHtml)
    (run-term WriteStrategy StrategiesDbId WriteTid WriteHtml _))
```

The `run-term WriteStrategy …` line is the load-bearing one. It runs the helper once per `(WriteTid, WriteHtml)` row of the iteration, with a single-row solution each time. This is mandatory whenever `add-or-get-by-name` is in the path — see [add-or-get-by-name binds the same block across multiple solution rows](../gotchas/add-or-get-by-name-multi-row.md). A direct cross-row invocation would bind one shared `Block` for every row and produce `KEY_UPDATE_CONFLICT` the moment two rows reach the per-row write.

### How to choose

| Question | Shape A — column write | Shape B — HTML block write |
|---|---|---|
| Is `content` plain text or rich HTML? | plain text (column) | rich HTML (rendered block) |
| Does PK alone identify the row uniquely? | yes — write all rows in one `write-table` | yes, but each row also needs a per-row Block under its row page |
| Multiple rows per natural key? (e.g. 2-4 drafts per treatment) | yes — composite PK like `[treatment_id, draft]` works | usually one row per natural key |
| `add-or-get-by-name` in the write path? | no — shape never touches Content blocks | yes — drives the per-row `run-term` requirement |
| Batch size | one call, all rows | one `write-table` + N per-row `run-term` invocations |

The decision is driven by the **shape of `content`**, not by row count or PK design. If the column can hold the value as a plain string, Shape A is simpler, faster, and avoids the multi-row hazard entirely. The moment `content` becomes an HTML payload addressed by Block name, Shape B is the only correct option — and the `add-or-get-by-name-multi-row` gotcha forces the `run-term` framing.

### Forecasting future agents

- **Critic** writes review-by-treatment in plain JSON-as-text or short scores → likely Shape A (column write).
- **Analyst** writes summary-by-treatment with rich-HTML rationale → likely Shape B (HTML block).
- **Execute** writes send-result-by-treatment in plain text → Shape A.

The wrong choice corrupts the contract on the first run. Cargo-cult the right exemplar (Strategist for Shape B, Copywriter for Shape A) before writing the next agent step.

## Pattern 3 — agent-scoped Memory cold-start gate

**Discovery context.** Every agent writes one Memory row per invocation, keyed `[agent, revision]`. The new revision is computed from the count of the agent's existing Memory rows. An empty Memory needs a different code path (revision = `"0"`) than a populated Memory (revision = `string-format "%d" Count`). The first MAB agent to write Memory was Researcher; later agents inherited the cold-start gate and the gate **silently broke**.

### Why the any-row guard works for the first writer and fails for everyone else

The original guard is "any Memory row exists":

```oql
;; Researcher (or any first writer): the only Memory rows that could exist
;; are this agent's own. The any-row check is equivalent to a self-check.
(with-table-if (Qo.Public.OqlApi.Db.Prop/prop-vals-by-folder MemoryDbId AnyMemPid _)
  [MemoryDbId AnyMemPid MemRevCount]
  (run-term CountThisAgentMemory MemoryDbId MemRevCount)
  (= MemRevCount 0))
```

When Researcher runs against an empty Memory, the guard's then-branch counts zero self-rows; the else-branch fires `(= MemRevCount 0)`. Correct.

When Strategist runs after Researcher, Memory already has `researcher` rows. The guard's then-branch fires (any row exists), and `CountThisAgentMemory MemoryDbId MemRevCount` is invoked. But that helper folds over `strategist`-scoped rows, of which there are zero — `fold` over zero solutions does **not bind** its accumulator (see [reducers.md § fold over zero solutions](../reference/reducers.md#fold-over-zero-solutions-does-not-bind-its-accumulator)). The helper no-returns, the surrounding clause silently produces no solutions, and the entire agent step skips its Memory write with no error trail.

The failure surfaces as "agent ran but no Memory row appeared" — diagnostically empty.

### The fix — agent-scope the guard predicate itself

The guard must be "any row **for this agent** exists":

```oql
;; Strategist / Copywriter / Critic / Analyst / Execute — agent-scoped guard.
(:- (WriteCopyMemoryRow Ctx CopyMemHtml MemoryDone {"capture" [CountCopywriterMemory CopywriterWriteMemory]})
    (get Ctx "memory-db-id" MemoryDbId)
    (with-table-if (and (Qo.Public.OqlApi.Db.Prop/prop-vals-by-folder MemoryDbId AnyMemPid AnyMemPv)
                        (get-in AnyMemPv ["prop-vals" "agent"] "copywriter"))
      [MemoryDbId AnyMemPid AnyMemPv MemRevCount CountCopywriterMemory]
      (call CountCopywriterMemory MemoryDbId MemRevCount)
      (= MemRevCount 0))
    (string-format "%d" MemRevCount NewMemRev)
    (call CopywriterWriteMemory MemoryDbId NewMemRev CopyMemHtml MemoryDone))
```

Two pieces matter:

1. The `with-table-if` condition is a compound `and` that checks for any Memory row **and** filters its `agent` prop-val to `"copywriter"` (the running agent's name). The then-branch fires only when at least one self-row exists.
2. The else-branch binds `MemRevCount` to `0` directly — never invoking the count helper.

When Copywriter cold-starts after Strategist, the `and` predicate finds zero `copywriter` rows even though `researcher` and `strategist` rows exist. The else-branch fires; revision becomes `"0"`; Memory write proceeds. Correct.

### Why the symptom hides

The any-row failure mode is invisible at design time because it depends on **deployment order** between agents. A test environment where Strategist runs against a Memory that is fully empty (no Researcher rows yet) will pass — the any-row guard correctly flags the cold start. A staging environment where Researcher has run once will pass too — Strategist's any-row guard fires the then-branch, the count is zero, the fold no-returns, and the step silently skips its write. The bug only manifests as "agent ran but Memory has no agent-scoped row," and the trail upstream of the empty fold is silent.

The fix is mechanical and uniform — every agent that writes Memory must use the agent-scoped guard. It is not optional for agents 2..N. Strategist's discovery #6 in the MAB session-handoff was the first capture of the rule; it generalises to every workflow agent step.

### Generalising

Any agent step that has its own per-agent slice of a shared multi-tenant table (Memory, Reports, anything keyed `[agent, …]`) and computes a next-id from row count must agent-scope its existence guard. The pattern is:

```oql
(with-table-if (and (prop-vals-by-folder DbId AnyPid AnyPv)
                    (get-in AnyPv ["prop-vals" "agent"] "<this-agent>"))
  [DbId AnyPid AnyPv NextId CountThisAgent]
  (call CountThisAgent DbId NextId)
  (= NextId 0))
```

The else-branch's literal `0` avoids the `fold over zero solutions` no-return entirely.

## Pattern 4 — LLM instruction specificity for audit/verification roles

**Discovery context.** The Format-phase LLM in the MAB Analyst step was instructed to "re-validate accepted drafts against hard rules implied by the Brief." It faithfully produced a Critic section asserting "All accepted drafts include working URLs" — without re-reading a single copy body. The dominant signal in its context was the Critic's `decision=accept` column, which it summarised rather than treating as an input to re-check.

### Why high-level audit instructions fail

An LLM has multiple signals in context at once: structured JSON output from prior steps, prose instructions, and example outputs. When the instruction says "re-validate" but the input contains a pre-computed verdict (e.g., `decision=accept`), the LLM will summarise the verdict. The verdict is a higher-quality signal than raw data — it was produced by a dedicated prior step — so the LLM treats it as the answer, not as a data column to audit against independently.

The failure is silent. The LLM produces a well-structured output that looks like it performed validation, because it did perform it — against the downstream signal rather than the upstream source. No parse error, no no-return. The output is simply sourced from the wrong layer.

### The four-part specificity rule

To force primary-source validation, the system prompt must specify all four of the following:

**1. The exact data structure to iterate.**

Wrong: `"re-validate accepted drafts against the hard rules"`

Right: `"for every draft where decision=accept in the critiques array"`

The LLM must be told which key in which object to filter on. Without the key name, it will iterate whatever feels most natural — typically the verdict column it already has.

**2. The exact field to read from source.**

Wrong: `"check the copy content"`

Right: `"read the content field from the copy array for that treatment_id+draft pair"`

The LLM must be told the field name (`content`), the table it lives in (`copy array`), and how to key into it (`treatment_id+draft pair`). Without this, "check the copy content" is satisfied by reading `critiques[].decision` — that IS copy-related content in the LLM's view.

**3. The exact condition to check.**

Wrong: `"verify URLs are present"`

Right: `"check each email object for presence of http:// or https://"`

The LLM must be given a concrete boolean test — a literal string to grep for, a field name to compare, a value to match. "Verify URLs" is ambiguous. The `http://` / `https://` check is not.

**4. The expected failure output.**

Wrong: `"report any violations"`

Right: `"if no URL found in any email, name the draft and the failing check"`

Without a failure output template, the LLM will surface only the summary verdict if all checks pass (which they appear to, because it sourced from the prior verdict column). The failure template forces the LLM to exercise the check against data — not to confirm the existing verdict.

### Example directive

This is the IMPORTANT paragraph added to the MAB Analyst Think-phase directive after the J.1 failure:

```
IMPORTANT: Re-validation means primary-source reading.
For every draft where decision=accept in the critiques array:
  1. Look up that draft's entry in the copy array by matching treatment_id and name.
  2. Read the content field. Iterate every email object in content.
  3. Check each email for presence of "http://" or "https://" anywhere in its body or subject fields.
  4. If no URL is found in any email for this draft, include a failing-check entry:
     {"draft": "<name>", "check": "url-presence", "result": "FAIL", "detail": "no http/https found in any email body or subject"}
  5. If every email has at least one URL, mark the check passed.
Do NOT infer URL presence from the Critic's decision=accept. Read the content field directly.
```

The five steps mirror the four-part rule exactly: step 1-2 = exact data structure + exact source field, step 3 = exact condition, step 4 = failure output template, step 5 = explicit instruction to not summarise from the verdict.

### Generalising

Any agent step where the LLM must audit, validate, or verify source data — not just synthesise conclusions from prior agents — must use field-level instruction specificity. The pattern is:

- **Iteration target**: name the key and the filter (`for every X where field=value in array`)
- **Source field**: name the table, the key, and the traversal path (`read field F from table T for the row matching K1=v1, K2=v2`)
- **Boolean test**: state the exact check, not the category (`contains http:// or https://`, not `has a URL`)
- **Failure template**: give the LLM a concrete failure record shape (`{"draft": ..., "check": ..., "result": "FAIL", "detail": ...}`)
- **Explicit non-inference**: state which prior output to NOT source from (`Do NOT infer X from the prior step's Y`)

The explicit non-inference line is the most frequently omitted. Without it, the LLM will source from the highest-quality signal it has — which is typically the upstream verdict, not the raw source data the audit is supposed to check.

### Forecasting future agents

Any agent step that:
- Has a "verification" or "quality check" role description in its spec,
- Receives both a structured verdict (decision, score, status) AND the raw data that verdict was derived from,
- And is instructed to "re-check" or "re-validate,"

…is a candidate for this failure. The spec instruction "re-validate X against Y" is not sufficient; all four parts must appear in the system prompt.

## Why these four patterns belong together

All four are encountered in the **write half** of an agent step — the phase after the LLM produces structured output (or must produce it by reading source data), before the implementation returns its Result envelope. They cohere as one shape because they share invariants:

- The LLM emits an array of records that must be validated for invariants the LLM didn't enforce → Pattern 1.
- The records must be batched into a primary table whose row shape determines the write strategy → Pattern 2.
- The agent records its reasoning into a per-agent slice of a shared Memory table whose existence guard must be agent-scoped → Pattern 3.
- When the LLM's role is audit or verification, the system prompt must specify the exact data path, field, condition, and failure output — otherwise the LLM will summarise from prior verdicts instead of re-reading source data → Pattern 4.

Cargo-culting one pattern without the others leaves a hole the next agent's first run will fall into. The four together form the minimum write-discipline kit for any new MAB-style agent step.

## Related

- [agent-step-plan-shape.md](agent-step-plan-shape.md) — the meta-discipline that produces the flat-sequential helper layout these patterns plug into. Read first when authoring a new agent-step plan.
- [reference/reducers.md § fold folds over unique values, not rows](../reference/reducers.md#fold-folds-over-unique-values-not-rows) — primitive-level home for "count source-array length, not the post-fold accumulator." Pattern 1 here names the agent-step shape; reducers.md names the language rule.
- [reference/reducers.md § fold over zero solutions does not bind its accumulator](../reference/reducers.md#fold-over-zero-solutions-does-not-bind-its-accumulator) — the failure mechanism that makes Pattern 3's any-row guard silently break for non-first agents.
- [gotchas/add-or-get-by-name-multi-row.md](../gotchas/add-or-get-by-name-multi-row.md) — the Block-UUID-collision hazard that drives the per-row `run-term` requirement in Pattern 2's HTML-block shape.
- [reference/clauses.md § Closures / Capture](../reference/clauses.md#closures--capture) — every helper in the exemplars declares its own `{"capture" [...]}`. Captures do not propagate through `call` / `run-term`; the rule applies to every level of the write-shape stack.
- [reference/control-flow.md](../reference/control-flow.md) — `with-table-if` semantics for the cold-start guards in Pattern 3.
